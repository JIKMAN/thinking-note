# 토스뱅크 DataOps Engineering 기술 면접 준비

## 기술 스택 매핑

| 토스뱅크 요구 기술 | 이력서 경험 | 매칭 강도 |
|---|---|---|
| Apache Spark (튜닝/최적화) | TravelWallet + ZoomInternet - 배치/스트리밍 파이프라인, 퍼포먼스 튜닝, SparkML | ★★★ |
| Apache Kafka | TravelWallet + ZoomInternet - 수집 파이프라인, 실시간 처리 | ★★★ |
| Apache Airflow | TravelWallet + ZoomInternet - 워크플로우 오케스트레이션 | ★★★ |
| Apache Iceberg | TravelWallet - 도입/운영, ZoomInternet - 마이그레이션 | ★★★ |
| Hadoop Ecosystem (HDFS/Hive/HBase) | ZoomInternet - 온프레미스 운영 및 클라우드 마이그레이션 | ★★☆ |
| Apache Flink | 경험 없음 | - |
| Trino | 직접 경험 없음 (ClickHouse OLAP 경험 유사) | ★☆☆ |

---

# 1. Apache Spark

## 아키텍처

```
[ Driver Program ]
  - SparkContext / SparkSession
  - DAG Scheduler: Job → Stage → Task 분해
  - Task Scheduler: 각 Task를 Executor에 할당

[ Cluster Manager ] (YARN / Kubernetes / Standalone)
  - Driver가 요청한 리소스(Executor 수, 메모리, 코어) 할당

[ Executor ]
  - Task 실행 단위 (JVM 프로세스)
  - On-Heap / Off-Heap 메모리 구분
  - Block Manager: 캐시, Shuffle 데이터 관리

[ 실행 흐름 ]
Transformation(lazy) → Action 호출 → Job 생성
→ DAG → Stage(Shuffle 경계) → Task(Partition 단위) → Executor 실행
```

**실행 계획 레이어:**
```
Unresolved Logical Plan
  → Analysis (Catalog 참조)
  → Logical Plan
  → Optimization (Catalyst - Predicate Pushdown, Column Pruning 등)
  → Physical Plan (여러 후보)
  → Cost-based Optimization (CBO)
  → 선택된 Physical Plan 실행
```

## 중요 개념

| 개념 | 설명 |
|---|---|
| **Lazy Evaluation** | Action 호출 시점에만 실제 연산 수행 |
| **Shuffle** | 네트워크를 통한 파티션 간 데이터 이동. 가장 큰 비용 |
| **Partition** | 병렬 처리 단위. 기본적으로 HDFS block 크기와 매핑 |
| **Broadcast Join** | 작은 테이블을 각 Executor에 복사 → Shuffle 없이 Join |
| **Sort-Merge Join** | 두 테이블 모두 큰 경우 사용. Shuffle 발생 |
| **Skew** | 특정 파티션에 데이터 쏠림 → 해당 Task만 느려짐 |
| **Caching** | `.cache()` / `.persist()` - Storage Level 설정 가능 |
| **Speculation** | 느린 Task를 다른 Executor에서 병렬 재실행 |

## 실무 주요 고려사항

**1. Shuffle 최소화**
- 불필요한 `groupBy`, `join` 순서 최적화
- Partition 수 조정: `spark.sql.shuffle.partitions` (기본 200)
- Broadcast Join 적극 활용: `spark.sql.autoBroadcastJoinThreshold`

**2. Data Skew 처리**
- Salting: 편향된 키에 랜덤 접두사 추가 후 처리
- `skewJoin` hint (Spark 3.x AQE)
- AQE(Adaptive Query Execution): 런타임에 파티션 병합/분할

**3. 메모리 설정**
```
spark.executor.memory=8g
spark.executor.memoryOverhead=2g  # Off-heap (Python, native 라이브러리용)
spark.memory.fraction=0.6         # Execution + Storage 영역
spark.memory.storageFraction=0.5  # Storage 영역 비율
```

**4. 파티션 전략**
- 입력 파티션: HDFS block = 128MB → 파티션 수 = 총 데이터 / 128MB
- 출력 파티션: 파일 크기 128MB~1GB 목표
- `coalesce` (셔플 없음) vs `repartition` (셔플 있음)

**5. Structured Streaming 고려사항**
- Trigger: 처리 주기 설정 (`processingTime`, `once`, `availableNow`)
- Checkpointing: 상태 복구를 위한 필수 설정
- Watermark: 늦게 도착한 데이터 처리 허용 범위

---

## 질문 + 모범답변

### 경험 검증 질문

**Q1. Spark 퍼포먼스 튜닝을 했다고 하셨는데, 실제로 어떤 문제가 있었고 어떻게 해결하셨나요?**

> A. TravelWallet에서 결제/유저 데이터를 처리하는 배치 파이프라인이 특정 시간대에 지연되는 문제가 있었습니다. Spark UI를 통해 Stage별 실행 시간을 분석해보니, Shuffle Read가 과도하게 발생하는 Stage가 병목이었습니다. 원인은 두 가지였는데, 첫째로 `shuffle.partitions`가 기본값 200으로 고정되어 있어 데이터 크기 대비 파티션이 너무 많았고, 둘째로 특정 유저 ID 기준으로 groupBy를 할 때 일부 파티션에 데이터가 집중되는 Skew가 발생하고 있었습니다. Shuffle Partition 수를 데이터 크기에 맞게 조정하고, AQE를 활성화하여 런타임에 Skewed Partition을 자동으로 분할하도록 설정해 처리 시간을 개선했습니다.

**꼬리질문: AQE를 활성화했을 때 항상 좋은 건 아닐 수도 있는데, 단점이나 주의사항은요?**

> A. AQE는 런타임 통계를 기반으로 동적으로 계획을 변경하기 때문에, 계획 재최적화 비용 자체가 발생합니다. 또한 반복적으로 사용하는 캐시된 데이터셋에 AQE를 적용하면 예상치 못한 재계획이 일어날 수 있어서, 캐시된 데이터셋은 AQE 적용 전에 충분히 테스트가 필요합니다. 그리고 Shuffle 파티션 병합(Coalesce)이 너무 aggressive하게 이뤄지면 이후 파티션이 너무 커져 OOM이 발생할 수도 있어서, `spark.sql.adaptive.coalescePartitions.minPartitionSize` 같은 파라미터를 함께 조정해야 합니다.

---

**Q2. Spark에서 Join 전략을 어떻게 선택하시나요? 실제로 Broadcast Join을 활용하신 경험이 있나요?**

> A. 기본적으로 한쪽 테이블이 Executor 메모리에 충분히 올라올 수 있는 크기라면 Broadcast Join을 사용합니다. TravelWallet에서 유저 세그먼트 배치를 처리할 때, 유저 메타 테이블(상대적으로 작은 dimension 테이블)과 대용량 트랜잭션 로그를 Join해야 하는 상황이 있었습니다. 이때 유저 메타 테이블을 broadcast hint로 명시하여 Shuffle 없이 처리해 성능을 개선했습니다. `spark.sql.autoBroadcastJoinThreshold`의 기본값(10MB)보다 큰 테이블이었는데, 실제 메모리 여유가 있어서 hint를 직접 지정했습니다.

**꼬리질문: Broadcast Join의 한계는 무엇인가요?**

> A. Driver가 broadcast할 데이터를 수집한 뒤 모든 Executor에 전송하기 때문에, 테이블이 클수록 Driver OOM과 네트워크 전송 비용이 문제가 됩니다. 또한 Executor 메모리가 부족한 상황에서 무리하게 Broadcast를 적용하면 Executor OOM이 발생할 수 있습니다. 그래서 실제로는 100MB~200MB 이하의 테이블에만 적용하는 것이 안전하고, 더 큰 테이블은 Sort-Merge Join을 사용하되 Bucketing으로 Shuffle을 줄이는 방식을 고려합니다.

---

**Q3. SparkML 파이프라인을 개발하셨다고 하셨는데, 어떤 방식으로 구성하셨나요?**

> A. TravelWallet 광고 플랫폼에서 타겟팅 유저 피처 생성을 위한 파이프라인을 구성했습니다. SparkML의 Pipeline API를 사용해 Feature Engineering 단계(StringIndexer, VectorAssembler, StandardScaler 등)와 모델 학습을 하나의 파이프라인으로 묶어 관리했습니다. 이렇게 하면 학습/추론 단계에서 동일한 전처리 로직이 보장되어 Feature Leakage를 방지할 수 있고, 파이프라인 자체를 직렬화해서 재사용하기도 편리합니다. 배치 주기로 유저 세그먼트를 재생성할 때 Airflow로 스케줄링하여 자동화했습니다.

---

### 공통 질문

**Q. 왜 Spark를 선택했나요? 다른 대안은 없었나요?**

> A. ZoomInternet에서는 이미 Hadoop 에코시스템 기반 온프레미스 환경이 있었기 때문에 자연스럽게 Spark가 선택됐습니다. TravelWallet에서는 0 to 1 설계 상황이었는데, 배치/스트리밍을 하나의 프레임워크로 처리할 수 있고, Python/SQL API가 풍부해 팀의 생산성과 유지보수 측면에서 유리하다고 판단했습니다. Flink도 검토했지만, 당시 팀의 규모와 스트리밍 요구사항 수준에서는 Spark Structured Streaming으로 충분했고, 러닝커브도 낮았습니다.

**Q. 토스뱅크처럼 금융 데이터를 대규모로 처리하는 환경에서 Spark 활용 시 가장 중요하게 생각해야 할 점은?**

> A. 세 가지를 가장 중요하게 봅니다. 첫째는 **데이터 정합성**입니다. 금융 데이터는 재처리 시에도 결과가 동일해야 하기 때문에 멱등성 설계가 필수입니다. Iceberg의 ACID 트랜잭션과 결합하면 중복 적재를 방지할 수 있습니다. 둘째는 **장애 복구**입니다. Checkpoint와 재처리 전략을 명확히 설계해야 합니다. 셋째는 **비용 효율**입니다. 대용량 배치 처리는 클러스터 리소스 비용이 크기 때문에, Partition 전략과 캐싱 전략으로 불필요한 재계산을 최소화해야 합니다.

---

# 2. Apache Kafka

## 아키텍처

```
[ Producer ] → [ Broker Cluster ] → [ Consumer Group ]
                    │
              [ ZooKeeper / KRaft ]
              (메타데이터, 리더 선출)

[ Topic ]
  └── Partition 0 : [Leader] [Follower] [Follower]  ← Replication
  └── Partition 1 : [Follower] [Leader] [Follower]
  └── Partition N : ...

[ Consumer Group ]
  - 하나의 Partition은 Group 내 하나의 Consumer만 소비
  - Consumer 수 > Partition 수: 일부 Consumer는 idle
```

**Producer → Broker 흐름:**
```
Producer → RecordAccumulator(배치) → Sender → Broker
  - acks=0: 응답 대기 없음 (유실 가능)
  - acks=1: Leader 수신 확인
  - acks=all(-1): ISR 전체 수신 확인 (가장 안전)
```

## 중요 개념

| 개념 | 설명 |
|---|---|
| **Partition** | 병렬 처리 단위. 한번 설정하면 줄이기 어려움 |
| **Offset** | Consumer가 읽은 위치. Group별로 독립 관리 |
| **ISR** | In-Sync Replicas: Leader와 동기화된 Follower 목록 |
| **Consumer Lag** | 최신 Offset - Consumer Committed Offset |
| **Compaction** | 같은 Key의 최신 값만 보존. CDC에 적합 |
| **Exactly-once** | Idempotent Producer + Transactional API 조합 |
| **Rebalancing** | Consumer Group 내 파티션 재할당. 처리 중단 발생 |

## 실무 주요 고려사항

**1. Partition 수 설계**
- 처리량 목표 / 단일 Consumer 처리량 = 최소 Partition 수
- 너무 많으면: Broker 메모리/파일 핸들러 부담, 리더 선출 지연
- 증설은 쉽지만 감소는 불가 → 초기 설계가 중요

**2. Consumer Lag 모니터링**
- Lag 급증 = Consumer 처리 속도 < Producer 발행 속도
- 원인: 처리 로직 병목, Consumer 수 부족, 파티션 Skew

**3. 재처리 전략**
- `auto.offset.reset`: earliest(처음부터) / latest(최신부터)
- Offset을 수동 커밋하는 경우 재처리 보장 가능
- Dead Letter Queue(DLQ) 패턴으로 실패 메시지 별도 처리

**4. 대용량 처리 시 배치 설정**
```
fetch.min.bytes: Consumer가 한번에 가져올 최소 바이트
fetch.max.wait.ms: 최소 바이트 미달 시 대기 시간
max.poll.records: 한번 poll에 가져올 최대 레코드 수
```

---

## 질문 + 모범답변

### 경험 검증 질문

**Q1. Kafka를 실제로 어떤 용도로 사용하셨나요? 파이프라인 구성을 설명해주세요.**

> A. 두 회사에서 다른 용도로 사용했습니다. ZoomInternet에서는 뉴스/블로그 포털의 대용량 로그(클릭, 노출, 검색 등)를 Kafka로 수집하고, Spark Streaming으로 소비해 HDFS에 적재하는 파이프라인을 운영했습니다. TravelWallet에서는 RDBMS의 변경 데이터를 Debezium으로 캡처해 Kafka 토픽으로 발행하고, Spark Structured Streaming으로 소비해 Iceberg 테이블에 UPSERT하는 CDC 파이프라인을 구성했습니다. 광고 플랫폼에서는 실시간 클릭/노출 이벤트를 Kafka로 발행하고 ClickHouse로 직접 소비해 실시간 집계하는 경로도 운영했습니다.

**꼬리질문: CDC 파이프라인에서 데이터 정합성을 어떻게 보장하셨나요?**

> A. 크게 세 가지로 보장했습니다. 첫째, Producer 쪽에서 Debezium의 at-least-once 전달을 전제로, Consumer(Spark)에서 중복 제거 로직을 적용했습니다. 소스 DB의 PK와 변경 타임스탬프를 기준으로 Iceberg의 MERGE INTO를 사용해 중복 이벤트를 멱등하게 처리했습니다. 둘째, Spark의 Checkpoint 기능으로 Consumer Offset을 관리해 장애 시 정확한 위치에서 재처리할 수 있도록 했습니다. 셋째, Iceberg의 스냅샷 격리를 활용해, 처리 중 실패 시 이전 커밋 상태로 롤백되도록 설계했습니다.

---

**Q2. Kafka Consumer Lag이 급증했을 때 어떻게 대응하시나요?**

> A. 먼저 Lag의 원인을 파악하는 것이 중요합니다. 모니터링에서 Lag 급증을 감지하면, Producer의 발행량이 증가했는지 vs Consumer의 처리 속도가 감소했는지를 먼저 구분합니다. 처리 속도 감소가 원인이라면 Consumer 로직의 병목 지점(외부 API 호출, DB 쿼리 등)을 확인합니다. ZoomInternet에서 운영하던 Slack 알림 파이프라인에서 Consumer Lag이 급증한 경험이 있는데, 당시 원인은 다운스트림 시스템의 응답 지연이었습니다. Consumer의 `max.poll.records`를 낮추고 배치 크기를 조정해 처리 시간 초과를 방지했고, 근본적으로는 다운스트림 호출을 비동기 처리로 개선해 해결했습니다.

---

### 공통 질문

**Q. 왜 Kafka를 선택했나요? 다른 메시징 시스템과 비교했을 때?**

> A. 대용량 로그/이벤트 처리에는 Kafka가 가장 적합했습니다. RabbitMQ 같은 전통적인 메시지 큐는 소비 후 메시지가 사라지지만, Kafka는 로그 보존 정책을 설정해 데이터를 일정 기간 재소비할 수 있습니다. 이는 파이프라인 장애 시 재처리와 다양한 Consumer(실시간/배치/분석)가 동일 데이터를 독립적으로 소비하는 구조에 매우 유리합니다. 또한 파티션 기반의 수평 확장이 자연스럽고, Spark/Flink/ClickHouse 등 대부분의 데이터 스택과 잘 통합됩니다.

**Q. 금융 데이터 처리 환경에서 Kafka 사용 시 가장 중요하게 생각해야 할 점은?**

> A. **메시지 유실 방지**와 **순서 보장**입니다. 금융 트랜잭션은 순서가 뒤바뀌거나 유실되면 데이터 정합성 문제가 생깁니다. acks=all로 설정해 ISR 전체 복제를 보장하고, Idempotent Producer로 중복 발행을 방지해야 합니다. 순서 보장이 필요한 경우 동일 계좌/유저의 이벤트는 반드시 같은 파티션에 들어가도록 파티션 키 전략을 설계해야 합니다. 그리고 Consumer의 오프셋 관리를 신중하게 해서, 처리 완료 후에만 커밋하는 수동 커밋 방식을 사용하는 것이 안전합니다.

---

# 3. Apache Airflow

## 아키텍처

```
[ Web Server ]  : UI, REST API
[ Scheduler ]   : DAG 파싱, Task 스케줄링, Executor에 태스크 전달
[ Executor ]
  - LocalExecutor : 단일 머신, subprocess
  - CeleryExecutor: 분산 Worker, Redis/RabbitMQ 브로커
  - KubernetesExecutor: Task마다 Pod 생성
[ Worker ]      : 실제 Task 실행
[ Metastore ]   : DAG 상태, Task 인스턴스, 변수, 커넥션 (PostgreSQL/MySQL)
[ DAG Files ]   : Python 코드로 정의된 DAG (공유 스토리지 또는 Git sync)
```

## 중요 개념

| 개념 | 설명 |
|---|---|
| **DAG** | Directed Acyclic Graph. Task 의존성 정의 |
| **Operator** | Task의 구현체 (BashOperator, PythonOperator, SparkSubmitOperator 등) |
| **Sensor** | 외부 조건이 충족될 때까지 대기 (FileSensor, S3KeySensor 등) |
| **XCom** | Task 간 소량 데이터 전달. 대용량 데이터는 외부 스토리지 사용 권장 |
| **Backfill** | 과거 특정 날짜 구간에 대해 DAG 재실행 |
| **execution_date** | 논리적 실행 시간 (실제 실행 시간과 다를 수 있음) |
| **Catchup** | Scheduler 재시작 시 누락된 과거 실행 자동 수행 여부 |
| **Pool** | 동시 실행 Task 수 제한 (리소스 관리) |

## 실무 주요 고려사항

**1. DAG 설계 원칙**
- 한 DAG = 하나의 명확한 비즈니스 목적
- Task 간 의존성은 데이터 의존성만으로 (불필요한 직렬화 지양)
- Idempotent Task 설계: 같은 날짜로 재실행해도 결과 동일

**2. 성능 & 안정성**
- DAG 파싱 부하: 파일 수, import 시간 최소화
- `max_active_runs`, `concurrency` 설정으로 과부하 방지
- SLA Miss 알림으로 지연 조기 감지

**3. 의존성 관리**
- `ExternalTaskSensor`: 다른 DAG의 Task 완료를 기다림
- Dataset 기반 스케줄링 (Airflow 2.4+): 데이터 이벤트 기반 트리거

**4. 비밀 관리**
- Variables, Connections는 Metastore 저장 → 암호화 권장
- AWS Secrets Manager / Vault 연동

---

## 질문 + 모범답변

### 경험 검증 질문

**Q1. Airflow로 대규모 워크플로우를 관리하신 경험이 있으신가요? 실제로 어떤 구조로 설계하셨나요?**

> A. TravelWallet에서 Airflow를 0부터 도입해서 전사 데이터 파이프라인 오케스트레이션을 구성했습니다. 파이프라인을 크게 수집(CDC/배치 적재), 가공(Iceberg DW 레이어 변환), 서빙(ClickHouse/유저 세그먼트) 레이어로 구분하고, 레이어 간 의존성은 ExternalTaskSensor로 연결했습니다. 각 DAG는 날짜 기반으로 멱등하게 실행되도록 설계해서, 장애 시 같은 날짜로 재실행해도 데이터 중복이 생기지 않도록 했습니다. Celery Executor로 구성해 Worker를 수평 확장 가능하게 하고, 각 도메인 파이프라인은 Pool을 나눠 특정 도메인 DAG가 리소스를 독점하지 않도록 관리했습니다.

**꼬리질문: Airflow 운영 중 겪었던 장애나 이슈가 있었나요?**

> A. Scheduler 프로세스가 DAG 파일 파싱 부하로 메모리가 증가해 느려지는 이슈가 있었습니다. 원인을 분석해보니 일부 DAG에서 모듈 import 시 외부 API를 호출하는 코드가 있어 파싱 자체가 느려지는 문제였습니다. DAG 파일 내 import 시점에 실행되는 코드를 제거하고, 무거운 초기화 로직은 Task 실행 시점으로 이동시켜 해결했습니다. 또한 DAG 파일 수가 늘어나면서 `min_file_process_interval`과 `dag_dir_list_interval` 설정을 조정해 파싱 주기를 최적화했습니다.

---

**Q2. Backfill은 어떤 경우에 사용하셨나요?**

> A. 두 가지 상황에서 주로 사용했습니다. 첫째는 신규 파이프라인 도입 시 과거 데이터를 소급 처리할 때입니다. TravelWallet에서 새로운 유저 세그먼트 기준을 도입할 때 과거 3개월치 데이터를 Backfill해야 했는데, 이때 동시 실행 수를 `max_active_runs`로 제한해 클러스터에 부하를 분산시키면서 처리했습니다. 둘째는 파이프라인 장애로 누락된 날짜가 생겼을 때입니다. Task가 멱등하게 설계되어 있어 특정 날짜만 선택적으로 재실행할 수 있었습니다.

---

### 공통 질문

**Q. 왜 Airflow를 선택했나요?**

> A. 배치 파이프라인 오케스트레이션 도구로 Airflow를 선택한 이유는 DAG를 Python 코드로 정의하기 때문에 버전 관리(Git)가 자연스럽고, 팀 내 코드 리뷰 프로세스에 통합하기 좋았습니다. 또한 SparkSubmitOperator, S3 연동, Slack 알림 등 풍부한 Provider 생태계 덕분에 외부 시스템 연동이 간편했고, UI에서 Task 상태와 실행 이력을 직관적으로 모니터링할 수 있어 운영 편의성이 높았습니다.

**Q. 금융 데이터 파이프라인에서 Airflow 사용 시 가장 중요하게 생각해야 할 점은?**

> A. **실패 감지와 재처리 보장**이 핵심입니다. 금융 파이프라인은 데이터 누락이 곧 비즈니스 오류로 이어질 수 있기 때문에, SLA Miss 알림과 Task 실패 알림을 반드시 구성해야 합니다. 재처리 시에도 결과가 동일하도록 모든 Task를 멱등하게 설계하고, Catchup 설정을 환경에 맞게 명확히 해야 합니다. 그리고 민감한 접속 정보(DB 크레덴셜, API 키)는 Connections와 Variables를 암호화하고, 필요하다면 외부 시크릿 관리 시스템과 연동하는 것을 권장합니다.

---

# 4. Apache Iceberg

## 아키텍처

```
[ Catalog ] (Hive Metastore / REST / AWS Glue / Nessie)
  └── 테이블 위치 정보, 현재 메타데이터 파일 포인터

[ Metadata Layer ]
  metadata.json
  └── snapshot 목록, schema, partition spec, sort order

  manifest-list (snapshot)
  └── manifest file 목록 + 파티션 통계

  manifest file
  └── data file 목록 + 행 수, 컬럼 통계, null 개수

[ Data Layer ]
  Parquet / ORC / Avro 파일 (S3, HDFS 등)
```

**핵심: Snapshot 기반 MVCC**
```
Write  → 새 Snapshot 생성 (기존 Snapshot 불변)
Read   → 특정 Snapshot을 기준으로 읽기
Commit → Catalog에 현재 Snapshot 포인터 원자적 업데이트
```

## 중요 개념

| 개념 | 설명 |
|---|---|
| **Snapshot Isolation** | 각 Write는 새 Snapshot 생성. 동시 Read/Write 가능 |
| **Schema Evolution** | 컬럼 추가/삭제/이름변경을 하위 호환성 있게 지원 |
| **Partition Evolution** | 기존 데이터 재작성 없이 파티션 전략 변경 가능 |
| **Hidden Partitioning** | 파티션 컬럼을 쿼리에 명시할 필요 없음 (자동 필터 적용) |
| **Time Travel** | 특정 snapshot ID 또는 timestamp 기준으로 과거 데이터 조회 |
| **Row-level Delete** | Position Delete / Equality Delete (V2 spec) |
| **Compaction** | 소규모 파일 병합, Delete 파일 병합으로 Read 성능 향상 |

## 실무 주요 고려사항

**1. 소규모 파일 문제 (Small File Problem)**
- 스트리밍/잦은 배치 적재 시 소규모 파일 다수 생성
- `rewrite_data_files` 프로시저로 정기 Compaction 필요
- Airflow로 Compaction DAG 스케줄링

**2. Delete 파일 관리**
- Row-level DELETE/UPDATE는 Delete 파일을 누적 생성
- Delete 파일이 많아지면 Read 시 병합 비용 증가
- `rewrite_position_delete_files`로 정기 정리 필요

**3. Catalog 선택**
- Hive Metastore: 기존 Hadoop 환경에 적합
- AWS Glue: AWS 환경에서 서버리스
- REST Catalog: 멀티 엔진(Spark/Flink/Trino) 환경에 유연
- Nessie: Git-like branching 지원

**4. MERGE INTO 설계**
```sql
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...
```
- CDC 파이프라인의 핵심 패턴
- 대량 MERGE는 Shuffle이 크기 때문에 배치 크기 조정 필요

---

## 질문 + 모범답변

### 경험 검증 질문

**Q1. Apache Iceberg를 도입하신 이유와 실제로 어떻게 활용하셨는지 설명해주세요.**

> A. TravelWallet에서 데이터 레이크를 설계할 때 결제, AML, 광고 등 다양한 도메인 데이터를 하나의 저장소에서 관리해야 했는데, 특히 소스 DB의 변경 데이터를 UPSERT 형태로 반영해야 하는 요구사항이 있었습니다. 기존 Hive 기반으로는 파티션 단위 덮어쓰기만 가능해 Row-level Update가 어려웠는데, Iceberg의 MERGE INTO를 활용하면 CDC 데이터를 효율적으로 처리할 수 있었습니다. 또한 Iceberg의 Schema Evolution 기능 덕분에 소스 DB 스키마 변경이 생겨도 기존 데이터 재작성 없이 컬럼을 추가할 수 있어 운영 부담이 줄었습니다. ZoomInternet에서는 온프레미스 Hadoop 환경을 S3 + Iceberg 기반 클라우드로 마이그레이션하는 작업을 했는데, 이때 Partition Evolution으로 기존 데이터 재처리 없이 파티션 전략을 개선할 수 있었습니다.

**꼬리질문: Iceberg에서 Compaction은 왜 필요하고 어떻게 관리하셨나요?**

> A. CDC 파이프라인처럼 잦은 Write가 발생하면 소규모 파일이 다수 생성됩니다. 파일이 많아지면 Reader가 metadata를 통해 수많은 파일을 열어야 하므로 Read 성능이 저하됩니다. 또한 Row-level DELETE/UPDATE가 많아지면 Delete 파일이 누적되어 Read 시 데이터 파일과 Delete 파일을 병합하는 비용도 커집니다. Airflow로 일별 Compaction DAG를 구성해 `rewrite_data_files`를 주기적으로 실행하고, 파일 크기 목표를 128MB~512MB로 설정해 관리했습니다. 테이블별 Write 빈도에 따라 Compaction 주기를 다르게 설정해 클러스터 부하를 분산했습니다.

---

**Q2. Iceberg의 Time Travel을 실제로 활용하신 경험이 있나요?**

> A. TravelWallet에서 파이프라인 버그로 특정 날짜의 데이터가 잘못 적재된 적이 있었습니다. Iceberg의 Time Travel 기능을 사용해 버그 발생 이전 Snapshot으로 과거 상태를 조회하고, 잘못 변경된 범위를 파악했습니다. 그리고 해당 Snapshot으로 롤백 후 파이프라인을 수정해 재처리했습니다. 이처럼 Snapshot 기반 구조는 장애 복구 시 데이터의 이전 상태를 안전하게 조회하고 복원할 수 있어 운영 안정성 측면에서 매우 유용했습니다.

---

### 공통 질문

**Q. 왜 Iceberg를 선택했나요? Delta Lake나 Hudi와 비교했을 때?**

> A. Iceberg를 선택한 가장 큰 이유는 **멀티 엔진 지원**과 **스펙의 명확한 분리**입니다. Delta Lake는 Databricks 중심 생태계에 최적화되어 있고, Hudi는 Spark에 강하게 결합되어 있는 편인데, Iceberg는 Spark, Flink, Trino, Presto, Hive 등 어떤 엔진에서도 동일한 테이블 포맷을 사용할 수 있어 엔진 의존성이 낮습니다. 또한 Partition Evolution과 Hidden Partitioning이 운영 관점에서 매우 편리했고, 메타데이터 레이어가 잘 설계되어 있어 대규모 테이블에서도 Plan 시간이 빠릅니다.

**Q. 토스뱅크처럼 대규모 금융 데이터를 Iceberg로 관리할 때 가장 중요하게 생각해야 할 점은?**

> A. 세 가지입니다. 첫째는 **Snapshot 관리**입니다. Snapshot이 무한정 쌓이면 메타데이터 파일이 커지고 Plan 비용이 증가합니다. `expire_snapshots`로 오래된 Snapshot을 정기적으로 정리해야 합니다. 둘째는 **Compaction 전략**입니다. 대규모 금융 데이터는 Row-level 변경이 빈번하기 때문에 Delete 파일 관리와 소규모 파일 병합을 체계적으로 관리해야 합니다. 셋째는 **Catalog 선택과 Concurrent Write 처리**입니다. 여러 파이프라인이 동시에 같은 테이블에 Write할 때 Optimistic Concurrency Control이 작동하는데, 충돌이 잦은 경우 Write 전략을 분리하거나 파티션을 나눠 충돌을 줄여야 합니다.

---

# 5. Hadoop Ecosystem (HDFS / Hive / HBase)

## 아키텍처

**HDFS:**
```
[ NameNode ]   : 메타데이터 관리 (파일→Block 매핑, DataNode 상태)
[ DataNode ]   : 실제 Block 저장 (기본 128MB, 복제 3)
[ Secondary NameNode / JournalNode(HA) ]: 메타데이터 체크포인트
```

**YARN:**
```
[ ResourceManager ] : 클러스터 전체 리소스 관리, ApplicationMaster 스케줄링
[ NodeManager ]     : 각 노드의 Container 실행 및 리소스 보고
[ ApplicationMaster ]: 개별 애플리케이션(Spark 등) 리소스 협상
```

**Hive:**
```
[ HiveServer2 ]   : JDBC/ODBC 인터페이스, 쿼리 수신
[ Metastore ]     : 테이블/파티션/스키마 메타데이터 (MySQL/PostgreSQL)
[ Execution Engine ] : MapReduce / Tez / Spark
```

**HBase:**
```
[ HMaster ]       : Region 할당, 부하 분산, DDL
[ RegionServer ]  : Region(행 범위) 관리, Read/Write 처리
[ ZooKeeper ]     : HMaster HA, RegionServer 상태 관리
[ Region ]        : MemStore(Write) + HFile(영구 저장) + WAL(복구)
```

## 중요 개념

| 개념 | 설명 |
|---|---|
| **HDFS Block** | 기본 128MB. 복제 수(기본 3)로 내결함성 확보 |
| **NameNode SPoF** | HA 구성(Active/Standby + JournalNode) 필수 |
| **HBase Row Key 설계** | Sequential Key → Hot Region 문제. Salting/Hashing 필요 |
| **Hive Partition** | 폴더 구조로 데이터 분리. 파티션 프루닝으로 쿼리 성능 향상 |
| **Hive Bucketing** | 파티션 내 버킷 분할. Bucket Join 최적화 가능 |
| **YARN Queue** | 파이프라인별 리소스 격리 및 우선순위 관리 |

## 실무 주요 고려사항

**1. HBase Row Key 설계 (Hot Region 방지)**
- Sequential Key(timestamp 등)는 하나의 Region에 집중
- Hashing 또는 Salting으로 Row Key 분산
- Scan 패턴과 Point Lookup 패턴을 분리 설계

**2. HDFS Small File 문제**
- 수많은 소규모 파일 → NameNode 메모리 부담
- Spark로 소규모 파일 병합 또는 S3/Iceberg로 마이그레이션

**3. YARN 리소스 관리**
- Capacity Scheduler / Fair Scheduler로 파이프라인 격리
- 배치와 스트리밍 파이프라인 큐 분리

---

## 질문 + 모범답변

### 경험 검증 질문

**Q1. HBase 스키마 설계 경험을 말씀해주세요. Row Key를 어떻게 설계하셨나요?**

> A. ZoomInternet에서 뉴스/블로그 포털의 실시간 데이터 서빙에 HBase를 사용했습니다. 주요 조회 패턴이 특정 유저의 최근 행동 이력 조회였기 때문에, Row Key를 `userId_reverseTimestamp` 형태로 설계했습니다. Timestamp를 역순으로 사용하면 최신 데이터가 Row Key 앞부분에 오기 때문에 Scan 시 최신 데이터를 빠르게 가져올 수 있습니다. 그리고 userId가 Sequential하지 않아 Hot Region 문제가 없었지만, 만약 Sequential한 ID를 사용해야 했다면 앞에 해시 prefix를 붙여 Region 분산을 유도했을 것입니다.

---

**Q2. 온프레미스 Hadoop 환경을 클라우드로 마이그레이션하신 경험을 말씀해주세요.**

> A. ZoomInternet에서 기존 온프레미스 HDFS + Hive 기반 데이터 레이크를 AWS S3 + Iceberg 기반으로 마이그레이션했습니다. 가장 어려웠던 점은 기존 ETL 파이프라인의 의존성이 복잡하게 얽혀 있어서 한 번에 전환하기 어려웠다는 점입니다. 그래서 단계적으로 접근했습니다. 먼저 신규 파이프라인부터 S3 + Iceberg로 적재하고, 기존 파이프라인은 Spark SQL 로직을 리팩토링하면서 단계적으로 전환했습니다. 쿼리 최적화도 함께 진행해 코드 복잡도를 줄이고 처리 성능도 개선했습니다. AWS Lambda로 서빙 레이어를 서버리스로 전환해 운영 부담도 줄였습니다.

---

### 공통 질문

**Q. 토스뱅크 환경(Hadoop Ecosystem 기반)에서 가장 중요하게 생각해야 할 운영 포인트는?**

> A. **클러스터 안정성과 리소스 격리**라고 생각합니다. 금융 데이터 파이프라인은 SLA가 엄격하기 때문에, 특정 파이프라인의 리소스 과소비가 전체 클러스터에 영향을 주지 않도록 YARN Queue로 배치/스트리밍/임시 쿼리를 격리하는 것이 중요합니다. 또한 NameNode는 HDFS의 단일 장애 지점이 될 수 있으므로 HA 구성과 메타데이터 백업이 필수입니다. 그리고 HBase를 사용한다면 Region 서버 부하 분산과 Compaction 정책을 잘 관리해야 지속적인 Write 성능을 유지할 수 있습니다.