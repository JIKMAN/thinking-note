1. 설계 의도 / 선택 근거
왜 Debezium 같은 표준 CDC 대신 JDBC polling?

- 소스 DB binlog 접근 권한 없음 (RDS 환경 제약)
- Kafka Connect 클러스터 별도 운영 부담
- 배치 주기로 충분한 SLA

Debezium 써보셨나요? 왜 안 썼나요?

- 알고 있음. Kafka Connect + Debezium 스택 운영 복잡도 대비 현재 요구사항엔 과함

binlog 기반 CDC의 장점을 알면서도 선택한 이유?

- 이벤트 단위 정합성 요구 없었음. 최종 상태 동기화로 충분

기술 부채라고 생각하지 않나요?

- 현재 SLA엔 적합. 실시간 요구 생기면 전환 로드맵 있음. 의도적 tradeoff


2. 정합성 / 데이터 품질
삭제된 레코드는 어떻게 감지?

- soft delete 컬럼(is_deleted, deleted_at) 기준 처리

soft delete가 없는 테이블은?

- 소스 테이블과 주기적 full reconciliation (예: 일 1회 PK 집합 비교)
없는 PK는 Iceberg에서 DELETE 처리

삭제 감지 못하면 zombie 레코드 남지 않나요?

- full reconciliation 주기만큼 lag 존재. 비즈니스 허용 범위 내로 SLA 협의

updated_at 기준 시 같은 타임스탬프 여러 건 누락 가능성?

- overlap window 적용 (마지막 성공 시각 - N분)
중복은 MERGE INTO upsert로 처리

타임스탬프 정밀도가 초 단위면?

- overlap window를 넉넉하게 (예: -10분) 잡고 MERGE로 흡수

DB 서버와 Spark 서버 시간 동기화 안 맞으면?

- 소스 DB의 NOW() 기준으로 watermark 계산. Spark 서버 시각 사용 안 함

polling 주기 사이에 insert→update→delete 모두 일어난 레코드는?

- 중간 상태 유실 인정. 최종 상태만 반영. 이벤트 단위 아닌 상태 동기화 방식
이벤트 단위 필요하면 binlog 기반으로 전환 필요


1. 성능 / 부하
소스 DB에 직접 JDBC로 붙으면 운영 DB 부하?

- Read replica 대상으로 연결
- 트래픽 피크 타임에도 같은 방식?
- 피크 타임 회피해서 스케줄링 (새벽 배치 등)

Read replica 쓰면 replication lag은?

- lag 감안해서 watermark에 여유 추가. SLA 협의 시 반영

파티셔닝 컬럼 인덱스 없으면 full scan 아닌가?

- PK 또는 인덱스 있는 컬럼만 파티셔닝 기준으로 사용

파티셔닝 컬럼 선정 기준?

- 숫자형 PK 또는 인덱스된 timestamp 컬럼
- 범위 분할이 균등하게 되는 컬럼 선택

numPartitions 값은 어떻게 정했나요?

- 테이블 크기 / 파티션당 적정 크기(예: 128MB) 기준 계산
- executor core 수와 맞춰서 병렬도 최적화

테이블 크기 커지면 polling 쿼리 느려지지 않나요?
- incremental 방식이라 full scan 아님. watermark 이후 범위만 쿼리
인덱스 있는 컬럼 기준이라 range scan

어떻게 스케일 대응?

numPartitions 늘려서 병렬도 증가
증분 범위가 너무 크면 micro-batch로 쪼개서 실행


1. 래그 / 실시간성
polling 주기가 얼마나 되나요?
- 테이블마다 다름. 중요도 따라 5분~1시간

그 주기 안에 비즈니스 요구사항 충족되나요?
- 현재 분석/배치 목적이라 충족. 실시간 알림 등은 별도 처리

실시간에 가까운 요구가 생기면?
- Debezium + Kafka 기반으로 전환. Iceberg 타겟 레이어는 그대로 유지 가능

이 tradeoff를 비즈니스 팀에 어떻게 설명했나요?
- "데이터는 최대 N분 지연될 수 있다"는 SLA로 명시. 실시간 필요한 지표는 원천 DB 직접 조회로 분리


5. 중복 / 멱등성
Spark job 실패 후 재실행 시 중복 안 들어가나요?

Iceberg MERGE INTO (upsert). PK 기준으로 MATCHED시 UPDATE, NOT MATCHED시 INSERT

Iceberg MERGE INTO 쓰나요? upsert 키는?

소스 테이블 PK 사용

exactly-once 어떻게 보장?

엄밀한 exactly-once는 아님. MERGE INTO로 idempotent하게 처리. at-least-once + dedup

overlap window 구간 중복은?

MERGE INTO MATCHED 조건에서 덮어씌움으로 해결. 중복 행 발생 안 함


6. 운영 / 장애 대응
소스 DB 스키마 변경되면?

Iceberg schema evolution으로 컬럼 추가는 자동 흡수
컬럼 삭제/타입 변경은 Spark job 실패로 감지 → 알림 → 수동 대응

컬럼 추가/삭제 Spark job 자동 대응 되나요?

추가: Iceberg mergeSchema 옵션으로 자동 반영 가능
삭제/타입변경: 자동 대응 어려움. 스키마 drift 모니터링 필요

소스 DB 장애 나면?

Airflow task retry. 실패 구간은 다음 실행 시 overlap window로 재수집

마지막 성공 지점부터 재개 가능?

마지막 성공한 max(updated_at) 또는 max(id) 를 메타 테이블에 저장. 다음 실행 시 해당 값부터 시작

DAG 여러 인스턴스 동시 실행 시 중복 읽기?

max_active_runs=1 로 동시 실행 방지
추가로 Iceberg Pool로 동일 테이블 동시 write 차단


7. 확장성
테이블 수십 개로 늘어나면?

DAG factory 패턴. 테이블 메타 정보(테이블명, watermark 컬럼, PK 등) config로 관리
config 추가만으로 새 테이블 파이프라인 자동 생성

각 테이블마다 DAG 따로 만드나요?

아님. DAG factory로 동적 생성

동적으로 테이블 추가하는 구조인가요?

config(yaml 또는 DB 테이블)에 항목 추가하면 다음 DAG 파싱 시 자동 반영

나중에 binlog 기반으로 전환할 계획?

이벤트 단위 정합성 요구 생기면 전환. Iceberg 타겟 레이어는 동일하게 유지 가능하므로 전환 비용 최소화

전환 비용이 얼마나 된다고 보나요?

Kafka Connect + Debezium 클러스터 구성, 소스 커넥터 설정, 기존 Airflow DAG 제거. 타겟 Iceberg 레이어는 재사용 가능하므로 인프라 비용이 주된 비용


8. 비교 지식
Debezium 동작 방식 설명해보세요

DB binlog(MySQL: binlog, Postgres: WAL) 구독 → row-level 변경 이벤트를 Kafka topic으로 스트리밍. INSERT/UPDATE/DELETE 모두 이벤트로 캡처. schema registry로 스키마 관리

binlog vs JDBC polling 근본적 차이?

binlog: 이벤트 기반, 삭제 감지 가능, lag 최소, 소스 부하 없음
JDBC polling: 상태 기반, 삭제 감지 어려움, polling 주기만큼 lag, 소스 DB 쿼리 부하

Kafka Connect Source Connector 고려해봤나요?

알고 있음. JDBC Source Connector는 polling 방식이라 동일한 한계 존재. Debezium Connector가 binlog 기반으로 진정한 CDC

Change Data Feed 알고 있나요?

Delta Lake는 CDF 자체 지원 (변경 행 + 변경 타입 컬럼 제공)
Iceberg는 공식 CDF 미지원. snapshot diff API로 변경 감지는 가능하나 Delta 수준은 아님. 이것도 Iceberg 대신 Delta를 고려할 수 있는 포인트 중 하나