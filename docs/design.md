# 개별 DB 트랜잭션 설계

상태: 검토 초안. 아래는 개인 검증용 구성으로, 아직 실행하지 않았다.

## 1. 업무와 데이터

승인된 한 요청으로 Oracle·PG의 지정 행·컬럼을 변경한다. 합성 데이터는 `item_id`, `status`, `version`으로 한정한다. 두 DB의 초기값은 `ACTIVE, 0`, 요청값은 `PAUSED, 1`로 둔다.

검증 대상은 해당 요청이 변경하는 조건·컬럼이다. 두 DB의 전체 데이터가 같아야 한다는 뜻이 아니다. PG의 독립적인 다른 데이터는 비교 대상에 포함하지 않는다.

1차에는 개별 DB 트랜잭션의 정상 완료와 실행 중 예외 처리를 구현한다. 이후 커밋 구간의 실패와 외부 조회로 보장 범위를 확인한다. ‘한쪽 실패 시 항상 양쪽 미변경’과 ‘전체 성공 결정 전 노출 금지’는 별도 요구사항으로 판정한다.

## 2. 매니저와 서비스 관계

| 구성 요소 | 역할 |
| --- | --- |
| oracleDataSource | Oracle 연결 |
| oracleTxManager | Oracle DataSourceTransactionManager |
| oracleJdbcTemplate | Oracle DML 수행 |
| postgresDataSource | PostgreSQL 연결 |
| pgTxManager | PostgreSQL DataSourceTransactionManager |
| postgresJdbcTemplate | PostgreSQL DML 수행 |
| OracleChangeService | oracleTxManager를 선택한 요청 처리 |
| PgChangeService | pgTxManager를 선택한 PG 처리 |
| ChangeFacade | 트랜잭션 종료 후 결과 수집·실패 이력·알림 대역 호출 |

DataSourceTransactionManager 하나는 하나의 DataSource를 관리한다. 각각을 등록하면 각 DB의 트랜잭션을 관리할 수 있지만, 두 커밋을 하나로 결정하는 기능까지 생기는 것은 아니다. [Spring 공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html)

매니저 이름을 어노테이션에 명시하고 Repository가 같은 DB의 DataSource를 사용하도록 연결한다. 트랜잭션 메서드는 다른 Bean의 프록시를 통해 호출한다. 예외를 내부에서 삼켜 정상 반환하는 경로와 `rollbackFor` 설정도 확인한다. [Spring 트랜잭션 어노테이션](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

## 3. 검증용 메서드 배치

아래 배치는 새 개인 실험의 예시다. 회사의 실제 서비스 배치·커밋 순서라고 주장하지 않는다.

~~~java
// OracleChangeService Bean: 외부 프록시를 통해 진입
@Transactional(transactionManager = "oracleTxManager", rollbackFor = Exception.class)
public void change(ChangeCommand command) {
    oracleRepository.update(command);
    pgChangeService.change(command); // 별도 Bean의 프록시 호출
    failpoint.afterPgServiceReturned();
}

// PgChangeService Bean: PG 트랜잭션을 별도로 시작
@Transactional(transactionManager = "pgTxManager", rollbackFor = Exception.class)
public void change(ChangeCommand command) {
    postgresRepository.update(command);
    failpoint.beforePgMethodReturns();
}
~~~

업무 메서드에서 Connection.commit()을 직접 호출하지 않고 각 매니저에 완료 처리를 맡긴다.

이 배치에서 예상하는 정상 순서는 다음과 같다.

~~~text
Oracle TX 시작 → Oracle DML
  PG TX 시작 → PG DML → PG 서비스 정상 반환 → PG 커밋 완료
Oracle 서비스 정상 반환 → Oracle 커밋 완료 → 요청 성공 응답
~~~

이 순서는 구현 후 매니저·커넥션의 실제 완료 기록으로 확인한다. DML 실행 완료와 메서드 본문의 반환, 트랜잭션 프록시의 커밋 완료를 같은 시점으로 기록하지 않는다.

## 4. 예외 위치에 따른 처리

| 실패 위치 | 이 구성에서 확인할 결과 |
| --- | --- |
| Oracle DML 실행 중 | Oracle 롤백, PG 미호출, 양쪽 대상 미변경 |
| Oracle 실행 후 PG DML 오류 | PG와 Oracle의 미커밋 변경 롤백 |
| 양쪽 DML 후 PG 메서드 반환 전 Java 예외 | 예외를 두 프록시까지 전파해 양쪽 미커밋 변경 롤백 |
| PG 커밋 완료 후 Oracle 서비스에서 예외 | Oracle 롤백, PG 변경 잔존 여부 확인 |

기존 업무에서 확인한 것은 Java 예외를 이용한 실패 경로와 대상 값 미변경이다. 실제 PG SQL 오류·커밋 경계 결과는 새 실험에서 별도로 확인한다.

이미 커밋한 변경은 다른 트랜잭션의 rollback으로 취소되지 않는다. 따라서 예외가 발생했다는 사실만으로 양쪽 미변경을 판정하지 않고 각 DB의 커밋 여부와 종료 후 값을 함께 본다. [JDBC 트랜잭션 문서](https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html)

## 5. 관찰과 성공 판정

각 사례는 업무 트랜잭션에 참여하지 않는 별도 연결로 대상 행을 조회한다. 테스트 전체를 감싸는 자동 롤백이나 같은 커넥션의 미커밋 조회로 결과를 대신하지 않는다.

조회 실험은 READ COMMITTED 조건에서 다음 시점을 구분한다.

1. 양쪽 DML 실행 후 어느 DB도 커밋하기 전.
2. PG 커밋 완료 후 Oracle 완료·요청 응답 전.
3. 요청 종료 후.

두 번째 시점의 조회는 부분 커밋과 요청 완료 전 노출을 확인하는 자료다. 이 구성에는 영속적인 전역 성공 결정이 없으므로, 조회값이 일치했다는 결과만으로 앞서 요구한 단일 커밋 결정·결정 전 노출 금지를 입증하지 않는다. 각 요구사항을 충족·미충족·미검증으로 따로 표시한다.

격리 수준·타임아웃·커밋 순서·오류 코드를 함께 기록한다. 고정 sleep으로 시점을 추측하지 않고 명시적인 배리어를 둔다.

## 6. 실패 이력과 관리자 알림

ChangeFacade는 OracleChangeService 프록시가 예외로 종료한 뒤 별도 Bean의 Oracle 실패 이력 저장을 호출한다. 새 저장 메서드는 명시적으로 별도 Oracle 트랜잭션을 사용하도록 설계한다. 업무 롤백과 실패 이력 저장은 각각 검증한다.

이것은 새 실험의 설계다. 기존 운영 코드의 실패 로그 저장 트랜잭션은 아직 자료로 확인하지 못했다.

첫 커밋 후 실패라면 ‘양쪽 롤백 완료’로 기록하지 않고 관찰한 부분 반영 상태를 남긴다. 커밋 결과가 불명확한 오류는 확정 롤백으로 바꾸지 않는다. Oracle 자체가 불가하면 이력 저장도 실패할 수 있으므로 해당 결과를 보존한다.

Teams는 실제 메시지 대신 호출 내용·요청 ID를 남기는 대역을 사용한다.

## 7. 후속 비교

D-01~D-06 결과를 확인한 뒤 추가 요구사항에 필요한 방식을 결정한다. XA/JTA, Seata AT, TCC는 아직 선택하지 않은 후속 후보다. 보충 자료는 [이전 XA 설계](advanced-xa-design.md)와 [대안 비교](alternatives.md)에 보존한다.

새 실험 결과를 과거 운영 경험의 증거로 바꾸어 쓰지 않는다. 구현·실행 전에는 위 흐름을 성과가 아닌 계획으로 표시한다.
