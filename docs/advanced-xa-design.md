# 보충 자료: 이전 XA/JTA 설계안

상태: 미확정 후속 후보. 현재 1차 범위는 [개별 DB 트랜잭션 설계](design.md)다. 아래는 이전 제안을 보존한 자료이며 구현·실행된 결과가 아니다. 본문의 ‘이번 프로젝트’·‘선정’은 이전 XA 제안의 범위를 뜻한다.

# 설계

상태: 검토 초안. 아래 구성과 검증은 구현 예정이며 아직 실행하지 않았다.

## 1. 이번 프로젝트가 검증하는 것

승인된 한 변경 요청으로 Oracle·PG의 대상 행을 변경한다. 검증용 데이터는 `item_id`, `status`, `version`으로 한정한다. 예를 들어 초기값은 두 DB 모두 `ACTIVE, 0`, 요청값은 `PAUSED, 1`이다. 이 값은 설계용 합성 예시다.

검증할 불변식은 **거래가 종료되고 필요한 복구가 완료된 뒤, 해당 요청의 두 DB 변경이 모두 반영되거나 모두 미반영인 것**이다. 원자적 커밋은 두 DB의 모든 데이터를 항상 같게 만드는 기능이나, 서로 다른 DB를 따로 읽는 모든 관찰에 단일 시점의 스냅샷을 주는 기능으로 설명하지 않는다.

PG가 독립적으로 관리하는 다른 테이블·컬럼은 이 요청의 원자적 변경 대상에 포함하지 않는다. 같은 대상 컬럼을 이 경로 밖에서 독립적으로 변경한다면 그 쓰기 경로에도 소유권·버전·충돌 정책이 필요하다. XA만으로 업무 규칙이나 모든 외부 쓰기의 정합성이 해결되는 것은 아니다.

## 2. 개별 DB 트랜잭션 비교 구성

사용자는 기존 업무에서 Oracle용·PG용 DataSourceTransactionManager를 각각 등록했다고 확인했다. 아래 메서드 배치와 커밋 순서는 아직 실제 회사 코드로 확인한 것이 아니라 검증용 비교 구성이다. 첫 구현에서는 두 개별 DB 트랜잭션 매니저의 실행 중 예외와 커밋 구간을 분리해 재현하고, 개선 방식의 선정은 그 결과와 요구사항을 함께 보고 확정한다.

DataSourceTransactionManager는 단일 DataSource를 관리한다. 매니저를 두 개 선언하거나 두 어노테이션을 중첩해도 자동으로 하나의 분산 커밋 결정이 생기지는 않는다. [Spring 공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html)

비교 실험에는 서로 다른 Bean의 프록시를 통과하는 다음 구성을 사용한다. 이 구성은 학습용 예제이며 기존 회사 코드가 이와 같았다는 뜻이 아니다.

~~~java
// OracleService Bean
@Transactional(transactionManager = "oracleTxManager", rollbackFor = Exception.class)
public void change(ChangeCommand command) {
    oracleRepository.update(command);
    pgService.change(command); // 다른 Bean의 pgTxManager 트랜잭션
    failpoint.afterPgServiceReturned();
}

// PgService Bean
@Transactional(transactionManager = "pgTxManager", rollbackFor = Exception.class)
public void change(ChangeCommand command) {
    pgRepository.update(command);
    failpoint.beforePgMethodReturns();
}
~~~

비교할 두 경로:

1. PG 메서드가 완료되기 전에 Java 예외 발생: PG와 Oracle의 미커밋 변경이 롤백될 수 있다.
2. PG 메서드가 정상 반환해 PG 커밋을 완료한 뒤 Oracle 메서드에서 Java 예외 발생: Oracle은 롤백되지만 이미 커밋된 PG 변경은 남을 수 있다.

두 번째 경로를 독립 관찰 연결로 확인해 부분 커밋을 재현한다. 이 결과로 회사 구현에 같은 결함이 있었다고 단정하지 않는다. 기존 실무의 실제 Bean·DataSource·실행 순서를 별도로 확인해야 한다.

## 3. XA/JTA 구성

역할을 다음처럼 나눈다.

- XA: 트랜잭션 매니저와 DB 자원이 prepare·commit·rollback·recover를 주고받는 표준.
- JTA: Java의 트랜잭션 관리 API.
- Narayana: 전역 결정과 복구를 담당하는 JTA 구현체로 선정할 후보.
- Spring JtaTransactionManager: Spring 트랜잭션 경계를 JTA 구현체에 위임하는 어댑터.
- Oracle·PG의 XADataSource: 각 DB를 전역 트랜잭션에 참여시키는 자원.

DB 자원이 실제 전역 트랜잭션에 등록돼야 한다. 클래스명과 어노테이션만 바꾸는 것으로 완료했다고 판단하지 않는다. [Oracle 분산 트랜잭션 문서](https://docs.oracle.com/en/database/oracle/oracle-database/26/jjdbc/distributed-transactions.html), [Spring JtaTransactionManager](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html)

~~~mermaid
flowchart TD
    A["승인된 합성 변경 요청"] --> S["변경 서비스<br/>globalTxManager 트랜잭션"]
    S --> O["Oracle XA 자원"]
    S --> P["PostgreSQL XA 자원"]
    T["Spring JtaTransactionManager<br/>Narayana"] -. "자원 등록·prepare·commit·rollback" .-> O
    T -. "자원 등록·prepare·commit·rollback" .-> P
    T --> L["영속 트랜잭션 로그"]
    R["Recovery Manager"] --> L
    R -. "기록된 결정으로 복구" .-> O
    R -. "기록된 결정으로 복구" .-> P
~~~

업무 메서드의 예정 형태:

~~~java
@Transactional(transactionManager = "globalTxManager", rollbackFor = Exception.class)
public void change(ChangeCommand command) {
    oracleRepository.update(command);
    pgRepository.update(command);
    failpoint.beforeBusinessMethodReturns();
}
~~~

위 메서드는 두 Repository의 커넥션이 실제 XA 트랜잭션에 참여한다는 설정을 전제로 한다. 구현 검증에서 이를 추적한다. 업무 메서드에서 개별 Connection.commit()을 호출하지 않는다.

## 4. 실패 시점별 의미

| 시점 | 기대하는 처리 | 잘못된 해석 |
| --- | --- | --- |
| DML 실행 중 오류, 커밋 결정 전 | 전역 중단·롤백. 복구가 필요하면 롤백 결정을 완료 | 모든 예외가 같은 단계의 실패라고 설명 |
| prepare 중 참여자 거절 | 커밋을 결정하지 않고 참여 자원을 중단 | prepare 거절과 commit 중 응답 유실을 같은 것으로 처리 |
| 전역 커밋 결정 기록 후 일부 자원 응답 실패 | 커밋 결정을 유지하고 남은 참여자의 커밋을 복구 | 이미 커밋한 DB까지 일반 rollback으로 되돌린다고 주장 |
| JVM 종료 후 재기동 | 동일한 영속 기록과 자원 정보를 이용해 결정대로 완료 | 저장소를 지우고 새 기동한 것을 복구 실험으로 기록 |

‘한쪽이라도 오류가 나면 언제나 둘 다 롤백’으로 표현하지 않는다. 커밋 결정 후에는 결정의 이행과 복구가 검증 대상이다. 처리 결과를 확인할 수 없는 응답 유실은 확정 실패로 바꾸지 않고 복구·조회가 필요한 상태로 다룬다.

## 5. 복구의 전제와 관찰

- 두 DB 모두 XA로 참여시키고 비-XA 자원을 끼워 넣는 last-resource 구성과 구분한다.
- 트랜잭션 로그는 JVM 재기동을 넘어 보존한다.
- 코디네이터 식별자는 동시에 실행되는 인스턴스 사이에서 유일해야 하고, 같은 노드의 재기동 실험에서는 일관되게 복구할 수 있어야 한다.
- 두 자원의 복구 연결·권한·자원 식별을 설정한다.
- PG의 prepared transaction 기능을 활성화하고 실제 값을 기록한다. 검증용 설정값을 운영 튜닝 권고값으로 설명하지 않는다.
- 재기동 전후의 pending/prepared 상태, 남은 복구 기록, 최종 DB 값을 함께 확인한다.

Narayana는 트랜잭션 로그와 노드 식별을 설정하며 복구를 제공한다. PG의 prepared 상태는 세션과 분리돼 남고, 정리 전에는 잠금을 유지할 수 있다. [Narayana Spring 통합 문서](https://github.com/snowdrop/narayana-spring-boot), [PostgreSQL PREPARE TRANSACTION](https://www.postgresql.org/docs/current/sql-prepare-transaction.html)

로그 유실·관리자의 임의 heuristic 처리·복구 불가능한 자원 손실까지 자동 해결한 것으로 주장하지 않는다. 장애 중 대기와 자원 점유가 발생할 수 있으므로 원자성과 가용성의 비용을 함께 기록한다.

## 6. 비교 조건

개별 DB 트랜잭션 예제와 XA 예제는 같은 합성 업무 입력·대상 컬럼·초기값을 사용한다. 격리 수준, 드라이버, 스키마 제약, 타임아웃을 기록하고 차이가 있으면 결과와 함께 밝힌다. Narayana Transactional Driver의 기본 격리 수준이 비교 예제와 달라질 수 있으므로 설정과 실제 커넥션 값을 확인한다. [Narayana Spring 통합 문서](https://github.com/snowdrop/narayana-spring-boot)

DB 스키마 생성은 업무 트랜잭션 전에 수행한다. 실패 후 값은 트랜잭션에 참여하지 않은 새 관찰 연결에서 읽는다. 테스트를 감싸는 자동 롤백 트랜잭션이나 동일 커넥션의 미커밋 조회로 결과를 판정하지 않는다.

## 7. 실패 이력과 Teams 알림

업무 변경의 원자성과 외부 알림 전달은 별도 문제다. 초기 구현은 실제 Teams 대신 알림 기록용 테스트 대역을 사용한다.

실패 이력은 업무 트랜잭션 종료를 확인한 바깥 처리에서 별도 Oracle 저장으로 검증한다. 전역 결과가 확정되지 않았을 때는 ‘전체 롤백 완료’라고 기록하지 않는다. Oracle 자체가 접근 불가능하면 Oracle 실패 이력을 반드시 즉시 저장한다고 보장할 수 없으므로 저장 실패도 관찰 결과에 포함한다.

이 설계는 새 개인 실험의 예정 방식이다. 기존 실무 로그 저장 트랜잭션이 이와 같았다는 뜻이 아니다.
