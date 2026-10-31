# 실행 증거 보존 형식

현재 상태: 실행 결과 없음. 아래는 구현 후 사용할 양식이다.

실행마다 `evidence/runs/<실행시각>-<case-id>/`를 만들고 다음 자료를 함께 보존한다.

~~~text
environment.json
command.log
case.json
transaction-trace.jsonl
oracle-before.json
postgres-before.json
oracle-after.json
postgres-after.json
read-observation.json
failure-record-observation.json
assertions.json
~~~

## 환경과 트랜잭션 기록

environment.json에는 소스 커밋, JDK·Spring·JDBC·DB 버전, 격리 수준, 타임아웃, DataSource·매니저 식별과 서비스 배치를 기록한다. Git 커밋 전 실행이면 소스 스냅샷의 식별값과 미커밋 상태를 남긴다. 비밀번호나 실제 업무 데이터는 포함하지 않는다.

transaction-trace.jsonl에는 case ID·request ID, 매니저·연결·DB 식별, DML 시작/완료, commit 시작/완료, rollback, 예외 발생·전파, failpoint 위치를 남긴다. 매니저의 완료 처리와 DB 관찰을 연결한다. SQL 실행 성공을 커밋 성공으로 표기하지 않는다.

case.json에는 초기값·변경 요청·오류를 만든 방법을 적는다. Java 예외를 PG SQL 오류나 서버 장애로 표기하지 않는다.

read-observation.json에는 업무 처리 중 배리어·관찰 연결·격리 수준·DB별 조회값·관찰 시각을 기록한다. 업무 종료 후 최종 값과 구분한다.

failure-record-observation.json에는 업무 결과, Oracle 이력 저장의 트랜잭션 경계·결과, 관리자 알림 대역의 기록을 담는다.

## 판정 형식

실행 전 예시는 다음과 같다.

~~~json
{
  "case_id": "D-05",
  "status": "PLANNED",
  "source_commit": null,
  "observed_commit_order": null,
  "oracle_final": null,
  "postgres_final": null,
  "test_assertions_passed": null,
  "consistency_requirement_met": null,
  "single_commit_decision_requirement_met": null,
  "pre_decision_visibility_requirement_met": null,
  "notes": null
}
~~~

실행 후에는 실제 관찰값과 판정 이유를 채운다. 부분 커밋을 예상대로 발견한 테스트의 통과와 정합성 요구 충족을 별도로 기록한다. 확인하지 않은 요구사항을 통과로 채우지 않는다.

## 후속 실험 기록

XA 등 분산 조정 방식을 별도로 구현하면 자원 참여·결정·prepare·recover 기록과 재기동 관찰 파일을 추가한다. 해당 실행에서만 코디네이터 버전·노드 식별·영속 로그·prepared 상태를 기록한다. 1차 개별 DB 트랜잭션 구성에 전역 결정이 있는 것처럼 형식을 채우지 않는다.
