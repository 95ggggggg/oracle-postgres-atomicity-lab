# ADR 0001 — 후속 후보: 두 독립 DB의 원자적 커밋에 XA/JTA 사용

상태: 미확정 제안. 1차 구현 범위는 개별 DB 트랜잭션 관리와 검증이다. 아래 내용은 후속 비교 자료이며 채택된 결정이 아니다.

추가 검토: 사용자가 비XA 대안을 질문하고 전체 성공 결정 전 변경값 노출 금지를 확인했다. [Seata AT·TCC·Saga 비교](../alternatives.md)를 추가했다. 아래 XA 제안은 아직 확정·구현된 결정이 아니다.

## 요구사항

한 승인 요청에서 Oracle·PG의 지정 변경을 하나의 커밋 결정으로 묶는다. 사용자에게 Oracle 우선 저장 후 PG 비동기 반영 방식을 제안했지만, 기존 요구사항을 유지하는 것으로 확인했다.

## 후속 선택안

두 DB의 XA 자원과 JTA 코디네이터를 사용한다. Spring 측 경계는 JtaTransactionManager로 연결한다. 구현체는 Narayana를 우선 검토한다. 직접 2PC 코디네이터를 작성하지 않는다.

## 대안 비교

| 접근 | 보장과 한계 | 이번 선택 |
| --- | --- | --- |
| 단일 DataSourceTransactionManager | 단일 DataSource의 개별 DB 트랜잭션 | 비교·설명 대상 |
| 두 개별 DB 트랜잭션 매니저의 순차 완료 | 커밋 전 오류는 양쪽 롤백될 수 있어도 부분 커밋 가능 | 한계 재현용 |
| ChainedTransactionManager | 부분 커밋 가능. 공식 문서에서 deprecated이며 불일치를 수용·복구할 수 있는 경우로 제한 | 새 해결책으로 채택하지 않음 |
| XA/JTA | 두 자원의 커밋 결정을 조정. 영속 기록과 복구 운영 필요 | 요구사항에 맞는 표준 접근으로 제안 |
| Outbox·CDC·멱등 소비 | 원본 커밋 뒤 다른 저장소를 반영하는 설계에서 유용. 지연·재처리·순서 관리 필요 | 일시적 차이를 허용하는 요구사항에서 검토 |
| 보상 처리 | 이미 커밋한 업무를 별도 업무로 복구. 원래 트랜잭션의 롤백과 다름 | 이번 원자적 커밋의 대체로 사용하지 않음 |

개별 DB 트랜잭션 매니저와 ChainedTransactionManager의 범위는 [Spring DataSourceTransactionManager](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html), [ChainedTransactionManager](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html)에 명시돼 있다. Outbox는 [AWS 설계 가이드](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)를 참고했다.

## 비용과 제한

XA를 선택하면 prepare·커밋 결정·완료·복구를 운영해야 한다. 장애 중 prepared 자원이 잠금을 유지하고 요청이 대기할 수 있다. 단순히 예외를 catch해 처리할 때보다 구성과 관찰 비용이 커진다.

전체 DB에 대한 동일 시점 읽기, 업무 규칙의 정확성, 경로 밖의 독립 쓰기까지 이번 원자성 검증의 성과로 포함하지 않는다. 시험한 장애와 복구 전제를 구체적으로 적는다.

전 세계 사용률 1위라는 통계적 주장으로 선택하지 않는다. 명시된 요구사항과 표준 프로토콜·운영 복구 지원을 기준으로 선택한다.
