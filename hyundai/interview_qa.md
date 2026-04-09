# 현대 데이터 엔지니어 면접 Q&A

---

## Q1. 트래블월렛에서 Apache Iceberg를 도입하셨는데, 왜 Iceberg를 선택하셨나요? Hudi나 Delta Lake는 왜 선택하지 않으셨나요?

**A.**
- 당시 요구사항: UPSERT 기반의 트랜잭션 처리 + 스키마 진화 + 파티션 관리 자동화
- Iceberg 선택 이유: Spark/Flink 멀티 엔진 지원, hidden partitioning으로 파티션 프루닝 자동화, time travel 기능으로 감사 추적 가능
- Hudi 배제 이유: Copy-On-Write / Merge-On-Read 설정 복잡도, 당시 Flink 통합 성숙도 낮음
- Delta Lake 배제 이유: Databricks 종속성 우려, AWS 환경에서 오픈소스 생태계 활용도 낮음

---

### Q1-1. Iceberg의 hidden partitioning이 구체적으로 어떤 문제를 해결해줬나요? 기존 Hive partitioning과 비교해서 설명해주세요.

**A.**
- Hive 파티셔닝: 쿼리에 WHERE dt='2024-01-01' 명시 필요 → 파티션 컬럼 노출, 휴먼 에러 발생
- Iceberg hidden partitioning: 메타데이터 레이어에서 파티션 정보 관리 → 쿼리에 파티션 컬럼 명시 불필요, 엔진이 자동으로 파티션 프루닝 수행
- 실무 효과: 파티션 컬럼 변경 시 쿼리 수정 없이 스키마 마이그레이션 가능

---

### Q1-2. Iceberg를 운영하면서 Small Files Problem은 어떻게 대응하셨나요?

**A.**
- 원인: 스트리밍/배치 빈번 커밋 시 small parquet 파일 누적
- 대응: Iceberg의 `rewriteDataFiles` + `expireSnapshots` 액션을 Airflow DAG로 주기적 실행
- 파일 타겟 사이즈 설정: `write.target-file-size-bytes` 조정 (128MB ~ 256MB)
- Snapshot 관리: `min-snapshots-to-keep`, `max-snapshot-age-ms` 정책 수립

---

### Q1-3. Iceberg에서 UPSERT(Merge Into) 처리 시 데이터 정합성을 어떻게 보장하셨나요? 동시성 문제가 발생한 경우는 없었나요?

**A.**
- Iceberg OCC(Optimistic Concurrency Control) 활용: 커밋 시점에 충돌 감지, 재시도 전략 적용
- 파이프라인 설계: 동일 파티션에 동시 쓰기 방지를 위해 Airflow DAG 의존성으로 직렬화
- 멱등성 보장: 배치 재처리 시 동일 결과 보장을 위해 `overwrite` 모드 + 파티션 단위 원자적 교체 방식 사용

---

## Q2. Spark 퍼포먼스 튜닝을 하셨다고 하셨는데, 실제로 어떤 병목을 발견하고 어떻게 개선하셨나요?

**A.**
- 주요 병목: Data Skew (특정 파티션에 데이터 쏠림), Shuffle Spill (메모리 부족으로 디스크 I/O 증가)
- 진단 도구: Spark UI의 Stage/Task 탭에서 task 실행 시간 편차 확인, GC time 모니터링
- 개선: Skew 대응 → salting 기법으로 키 분산 / `spark.sql.adaptive.skewJoin.enabled=true` (AQE)
- Shuffle 최적화: `spark.sql.shuffle.partitions` 조정, broadcast join 적용 가능 테이블 식별

---

### Q2-1. AQE(Adaptive Query Execution)를 실제로 사용하셨나요? 어떤 효과가 있었나요?

**A.**
- AQE 활성화: Spark 3.x에서 기본 활성화, skewJoin / coalescePartitions 자동 처리
- 효과: 런타임에 파티션 수 자동 조정 → 불필요한 small task 제거, shuffle read 감소
- 주의점: 복잡한 쿼리에서 AQE plan 재계산 오버헤드 발생 가능 → 통계 정확도(ANALYZE TABLE) 중요

---

### Q2-2. Spark 작업 중 OOM(Out of Memory)이 발생했을 때 어떻게 디버깅하고 해결하셨나요?

**A.**
- 원인 분석: Driver OOM vs Executor OOM 구분 → Spark UI에서 executor log 확인
- Driver OOM: `collect()`, `toPandas()` 등 전체 데이터 드라이버 수집 코드 제거 → 파일 직접 write로 전환
- Executor OOM: `spark.executor.memory` / `spark.memory.fraction` 조정, join 전 filter 추가로 데이터 줄이기
- Shuffle Spill: `spark.sql.shuffle.partitions` 늘려 파티션당 데이터 크기 감소

---

### Q2-3. Spark과 ClickHouse를 함께 사용하셨는데, 두 엔진의 역할을 어떻게 분리하셨나요?

**A.**
- Spark 역할: 대용량 배치 ETL, 복잡한 조인/집계, ML 피처 생성 → Iceberg 저장
- ClickHouse 역할: 전처리된 데이터의 실시간 집계 서빙, Materialized View로 지표 선계산
- 데이터 흐름: Kafka → Spark Structured Streaming → Iceberg (원천) → ClickHouse (서빙 레이어)
- 선택 기준: 쿼리 응답시간 요구사항 < 1초 → ClickHouse, 복잡도 높고 배치성 → Spark

---

## Q3. 멱등성 처리 로직을 표준화하셨다고 하셨는데, 구체적으로 어떻게 구현하셨나요?

**A.**
- 핵심 원칙: 같은 입력으로 동일 파이프라인 재실행 시 결과 동일 보장
- 구현: 파티션 단위 overwrite (Iceberg dynamic overwrite) → 해당 날짜 파티션만 교체
- 중복 제거: `row_number() OVER (PARTITION BY pk ORDER BY updated_at DESC) = 1` 로 최신 레코드만 유지
- 실행 ID 관리: Airflow의 `logical_date` 기반 실행 → 동일 날짜 재실행 시 항상 동일 결과

---

### Q3-1. Kafka에서 데이터를 소비할 때 exactly-once 처리를 어떻게 보장하셨나요?

**A.**
- Spark Structured Streaming + Kafka: offset 기반 체크포인트 관리
- Iceberg 연동: 커밋을 트랜잭션 단위로 묶어 at-least-once + 중복 제거로 exactly-once 의미론 구현
- 장애 복구: 체크포인트에서 마지막 성공 offset 재확인 후 재처리, Iceberg overwrite로 중복 방지

---

### Q3-2. 파이프라인 재처리(backfill) 시 어떤 전략을 사용하셨나요? 다운스트림에 영향을 최소화하는 방법은?

**A.**
- Airflow `backfill` 커맨드로 특정 날짜 범위 재실행
- 다운스트림 격리: 재처리 시 임시 테이블 또는 브랜치(Iceberg branching) 활용 → 검증 후 교체
- 순차 실행: `depends_on_past=True` 설정으로 직전 날짜 성공 후 다음 날짜 실행 보장
- 대용량 backfill: 날짜 범위를 작게 쪼개 병렬도 제한하여 운영 파이프라인 리소스 경합 방지

---

## Q4. 데이터 거버넌스 관련해서 메타데이터 관리를 어떻게 하셨나요? DataHub나 Amundsen 같은 도구를 사용해보셨나요?

**A.**
- 트래블월렛: 별도 카탈로그 도구 미도입, Iceberg 메타데이터 + Glue Catalog 조합으로 스키마 관리
- 줌인터넷: AWS Glue Data Catalog로 테이블 메타데이터 중앙화
- DataHub/Amundsen 직접 운영 경험은 없으나, Iceberg의 테이블 속성(description, owner 태그) 활용
- 개선 필요성 인지: 데이터 리니지, 오너십, 품질 지표를 통합 관리하는 카탈로그 도구 도입 필요성 체감

---

### Q4-1. 데이터 리니지(Lineage)를 추적할 필요가 있었던 상황이 있었나요? 어떻게 대응하셨나요?

**A.**
- 상황: 특정 지표 이상 탐지 시 어떤 소스 데이터가 원인인지 역추적 필요
- 대응: Airflow DAG 구조가 리니지 역할 수행 (upstream/downstream task 의존성 명시)
- 한계: 컬럼 레벨 리니지 추적 불가, 수작업 문서화 필요 → DataHub 도입 시 해결 가능 영역
- Spark의 `QueryExecutionListener` 활용한 SQL 기반 리니지 파싱 검토한 경험

---

### Q4-2. 데이터 품질 모니터링을 어떻게 구축하셨나요? 이상 탐지 시 어떤 흐름으로 대응하셨나요?

**A.**
- 모니터링 항목: row count 급감, null rate 임계치 초과, 처리 지연(SLA 미준수)
- 구현: Airflow task 내 품질 체크 로직 → 임계치 초과 시 task fail + Slack 알림 발송
- 알림 구조: 소스별 DAG에 품질 체크 task 추가 → 실패 시 자동 Slack 채널 알림 (webhook)
- 대응 프로세스: 알림 수신 → 소스 확인 → 파이프라인 재처리 또는 소스 팀 협업

---

## Q5. 온프레미스 Hadoop에서 AWS 클라우드로 마이그레이션을 하셨는데, 가장 어려웠던 부분은 무엇이었나요?

**A.**
- 가장 어려운 부분: 복잡한 ETL 의존성 파악 및 Spark SQL 리팩토링
- 기존 Hive 쿼리가 Hive 특화 UDF, SerDe 사용 → Spark SQL 호환성 문제 해결
- 파티션 전략 재설계: HDFS 파일 구조 → S3 + Iceberg hidden partitioning 전환
- 검증 전략: 병렬 운영 기간 동안 온프레미스 결과와 클라우드 결과 수치 비교 검증 후 컷오버

---

### Q5-1. 마이그레이션 중 데이터 정합성을 어떻게 검증하셨나요? 원본 테이블은 계속 업데이트되는데 어떻게 동일하다고 비교할 수 있나요?

**A.**
- 핵심 전제: 실시간으로 변하는 소스를 "지금 이 순간" 비교하는 건 불가능 → **기준 시점(watermark)을 고정**하는 것이 핵심
- 방법: `updated_at < '{기준 시각}'` 조건으로 소스 스냅샷 범위 고정 → 해당 범위 내 데이터만 타겟과 비교
- 실무 적용: 배치 파이프라인 완료 시각을 watermark로 설정, 그 이전 데이터만 비교 대상으로 삼음
- 비교 지표: row count, 핵심 컬럼 집계값(SUM, COUNT DISTINCT), 샘플 레코드 hash 비교

---

### Q5-1-1. watermark를 기준으로 고정하면, 그 이후에 들어온 늦은 데이터(late data)는 어떻게 처리하셨나요?

**A.**
- Late data 발생 원인: 네트워크 지연, 소스 시스템 배치 지연, CDC lag 등
- 대응 전략: 파티션 단위 재처리 허용 구조 설계 → `updated_at` 기준으로 D+1, D+2 재처리 DAG 구성
- Iceberg 활용: 해당 파티션만 overwrite → 다운스트림 영향 없이 수정 가능
- 허용 범위 정의: SLA 기준 "최대 N시간 이내 데이터는 반영" 정책 수립 → 그 이후는 별도 예외 처리

---

### Q5-1-2. 소스 DB가 RDBMS인 경우, 대량 비교 쿼리를 소스 DB에 직접 날리면 운영 DB에 부하가 걸리지 않나요? 어떻게 대응하셨나요?

**A.**
- 문제: 정합성 검증용 `COUNT`, `SUM` 쿼리가 운영 DB에 직접 부하 발생
- 대응 1: **Read Replica** 활용 → 검증 쿼리를 replica에 분리
- 대응 2: CDC 기반 파이프라인이면 원본 DB 직접 조회 대신 **Kafka offset 기반 카운트** 비교
- 대응 3: 소스에서 집계값을 미리 추출해 메타 테이블에 저장 → 검증 시 메타 테이블과 비교 (소스 재조회 최소화)
- 실무: 트래블월렛에서는 Kafka를 소스로 쓸 때 consumer lag 모니터링으로 누락 여부 1차 확인 후, 배치 완료 시점에 타겟 집계값과 대조

---

### Q5-1-3. row count나 집계값이 일치해도 실제 데이터 내용이 다를 수 있지 않나요? 컬럼 레벨 정합성은 어떻게 검증하셨나요?

**A.**
- row count 일치 = 정합성 보장 아님 → 같은 건수지만 값이 다를 수 있음
- 접근 방법 1: **체크섬(Checksum) 비교** → 핵심 컬럼들을 concat 후 MD5/SHA 해시, 소스와 타겟 hash 값 비교
- 접근 방법 2: **샘플 레코드 조회** → 랜덤 또는 특정 pk 기반으로 소스와 타겟 raw 값 직접 대조
- 접근 방법 3: **컬럼별 통계 비교** → NULL 비율, MIN/MAX/AVG 등 분포 지표를 소스·타겟 간 비교
- 실무 한계 인정: 전수 검증은 비용 과다 → 핵심 컬럼(금액, 상태값 등)만 선별 검증 + 이상 시 드릴다운

---

### Q5-2. AWS Lambda로 데이터 서빙을 하셨는데, 고가용성과 레이턴시 측면에서 어떤 트레이드오프가 있었나요?

**A.**
- 장점: 서버리스로 운영 부담 없음, 트래픽 급증 시 자동 스케일링
- 단점: Cold Start 레이턴시 (최초 실행 시 100~500ms 지연)
- 대응: Provisioned Concurrency 설정으로 critical 엔드포인트 warm 유지
- 데이터 소스: Lambda → DynamoDB or ElasticCache 조회로 응답 속도 최소화, 집계 결과는 S3 pre-computed 파일 활용

---

## Q6. 현대자동차에서는 모빌리티 도메인 데이터를 다루게 됩니다. 차량 위치 데이터나 경로 데이터는 일반 트랜잭션 데이터와 어떤 차이가 있다고 생각하시나요?

**A.**
- 데이터 특성 차이: 시계열 + 공간정보(위경도) 복합 → 시간 순서와 공간 인덱싱 모두 필요
- 볼륨: 차량 수 x 수집 주기(초 단위) → 초당 수백만 이벤트 처리 필요 → Kafka + Flink 조합 고려
- 분석 패턴: 경로 재구성(GPS 보정), 체류지 분석, OD(Origin-Destination) 매트릭스 생성
- 저장 포맷: 시계열 특화 파티셔닝(device_id + date) + 공간 인덱스(Geohash, H3) 활용 검토

---

### Q6-1. 대규모 실시간 이벤트 데이터를 처리할 때 Spark Streaming과 Flink 중 어떤 상황에서 각각을 선택하시겠나요?

**A.**
- Spark Structured Streaming 선택: 기존 Spark 생태계와 통합, 마이크로배치로 충분한 레이턴시 허용(수 초), 팀 Spark 숙련도 높을 때
- Flink 선택: 이벤트 시간 기반 정확한 윈도우 처리 필요, 밀리초 단위 레이턴시 요구, 상태 관리 복잡도 높을 때 (ex. 차량 이상 감지 실시간 알람)
- 모빌리티 맥락: 차량 이상 감지, 실시간 경로 추적 → Flink / 일별 주행 통계 집계 → Spark

---

### Q6-2. GPS 데이터 특성상 노이즈(튀는 좌표값)가 많을 텐데, 데이터 품질 측면에서 어떻게 접근하시겠나요?

**A.**
- 이상값 탐지: 두 연속 좌표 간 이동 속도 계산 → 물리적으로 불가능한 속도(ex. 500km/h 초과) 필터링
- 보정 전략: Kalman Filter 또는 Map Matching 알고리즘으로 도로 위 좌표 보정
- 파이프라인 설계: raw 데이터는 원본 보존 (Iceberg), 품질 처리 후 curated 레이어 분리 저장
- 모니터링: 디바이스별 노이즈 비율 지표화 → 임계치 초과 시 알람 (하드웨어 이상 가능성)
