# 🎯 이력서 주요 업무별 딥다이브 면접 가이드 (5-Depth 꼬리질문)

면접관이 "진짜로 문제를 깊게 고민하고 해결했는지"를 검증하기 위한 매우 날카로운 실무 밀착형 질문 체계입니다. 핵심 키워드를 위주로 내재화하여 답변을 준비해 보세요.

---

## 1️⃣ [Iceberg] 전사 통합 데이터 레이크 구축 및 마이그레이션

**Q. 왜 다양한 오픈 테이블 포맷(Hudi, Delta 등) 중에서 굳이 Iceberg를 선택했으며, 기술을 선택하고 아키텍처를 설계하기까지 어떤 비즈니스적 고민이 있었나요?**
* **Depth 1:** 기존 하둡(Hive/HDFS) 환경에서 마주했던 가장 뼈아픈 비즈니스적 한계점(Pain Point)은 무엇이었습니까?
  * **A (키워드):** 정합성 깨짐, 결제/정산/유저 도메인의 빈번한 Update/Delete 처리 불가, HDFS 파일 레벨 락(Lock), 스키마 에볼루션(Schema Evolution) 대응 장애.
* **Depth 2:** 그렇다면 Delta Lake나 Apache Hudi에 비해 조직의 트래픽 패턴과 AWS 데이터 생태계에서 Iceberg가 더 낫다고 판단한 근거는 무엇인가요?
  * **A (키워드):** S3 Object Storage와 완벽한 호환성, 메타데이터 트리에 기반한 O(1) 파일 탐색(디렉토리 스캔 제거), Hidden Partitioning 지원, Spark/Trino 상호 운용성(Vendor-agnostic).
* **Depth 3:** Iceberg 테이블 설계 시 트랜잭션 처리를 위해 Copy-on-Write(CoW) 와 Merge-on-Read(MoR) 중 어떤 방식을 선택했고 트레이드오프 장단점은 무어였나요?
  * **A (키워드):** 데이터 성격별 분리. 잦은 업데이트(광고/트래픽)는 빠르고 가벼운 쓰기가 가능한 MoR. 야간 대규모 정산 배치는 읽기가 빠른 CoW 선택.
* **Depth 4:** (MoR을 썼다면) MoR 방식 사용 시 시간이 지날수록 Delete File이 쌓이면서 조회 성능(Read Latency)이 심각하게 저하되었을 텐데, 실무에서 이를 어떻게 방어하고 운영했나요?
  * **A (키워드):** 백그라운드 Compaction 스케줄링. Spark Maintenance Job (Rewrite Data Files / Rewrite Manifests / Expire Snapshots) 주기적 실행으로 파티션 파편화 해결.
* **Depth 5:** 데이터 레이크에서 완벽히 수용하기 힘든 '초실시간(Sub-second) 요구사항'은 결국 어떻게 타협했나요? 레이크하우스 아키텍처 전체의 비즈니스적 장점은 어떻게 정리할 수 있습니까?
  * **A (키워드):** 초실시간은 HBase/ClickHouse 서빙 레이어로 위임(Lambda 아키텍처 혼합). 레이크하우스(Iceberg)는 SSoT(단일 진실 공급원)로서 '결제-유저-광고' 데이터를 하나의 통합 뷰로 제공, 컴플라이언스(보안/권한) 거버넌스 비용 단축.

---

## 2️⃣ [Spark] 대규모 배치·실시간 파이프라인 구축 및 최적화

**Q. 다양한 소스(RDBMS, Kafka)의 데이터를 Spark로 안정적으로 처리했다고 하셨는데, 장애 발생 상황에서도 '정확히 한 번(Exactly-once) 혹은 멱등성'을 달성하기 위한 구체적인 아키텍처는 무엇이었나요?**
* **Depth 1:** 금융권/결제 서비스라 중복이나 유실이 치명적인데, Spark 구조 상에서 어떤 방식으로 멱등성을 보장했나요?
  * **A (키워드):** Spark Checkpoint(HDFS/S3), Sink 단(Iceberg/RDBMS)의 PK 기반 MERGE/UPSERT 처리(Idempotent Sink), Kafka 오프셋의 트랜잭셔널 커밋.
* **Depth 2:** 실무에서 네트워크 지연으로 CDC나 로그 데이터가 뒤섞여 들어오는(Out-of-Order) 문제가 발생하면 파이프라인 정합성은 어떻게 보상했나요?
  * **A (키워드):** Event Time 기반 Watermark 처리, Timestamp 및 Source Sequence Number를 비교하여 Iceberg/DB 기록 시 구버전(Stale) 레코드 무시 처리.
* **Depth 3:** Spark 파이프라인 운영 중 데이터 폭증으로 가장 심각하게 마주한 성능 저하 혹은 OOM(Out of Memory) 스토리를 들려주세요. 병목은 주로 어디였나요?
  * **A (키워드):** 대용량 광고/유저 데이터 조인(Join) 시 특정 키에 트래픽이 몰리는 Data Skewness 발현. 메모리 스필(Spill to Disk) 및 GC 한계로 엑스큐터(Executor) 터짐 발생.
* **Depth 4:** Execution Plan(실행 계획) 단에서 해당 이슈를 어떻게 디버깅하고 튜닝(Tuning)하여 해결했습니까?
  * **A (키워드):** Broadcast Hash Join 강제(힌트 활용), 파티션 해시 쏠림을 풀기위한 Salting 기법, Shuffle Partition 수 상향 조정, Executor/Memory Overhead 비율 조정.
* **Depth 5:** 그 튜닝과 모니터링 고도화를 거친 결과, 실무 부서(비즈니스) 측면에서 얻거나 증명한 수치적인 이익이나 장점은 무엇인가요?
  * **A (키워드):** 배치 SLA 마감 시간 준수(목표 시간 O분에서 O분 단축), 재처리로 인한 개발 리소스 낭비 80% 감소, 컴퓨팅 노드 오토스케일 비용 연간 OOO만원 절감.

---

## 3️⃣ [Kafka/Fluentd] 대규모 로그 수집 파이프라인 고도화 (줌인터넷)

**Q. 포털 서비스 급의 대규모 로그를 Kafka와 Fluentd로 수집할 때, 트래픽 폭증기능이나 장애 상황 대비를 어떻게 설계했나요?**
* **Depth 1:** 서버 트래픽 폭발 시 앞단의 Fluentd가 부하를 견디지 못하고 로그를 버리는(Drop) 상황을 막기 위해 어떤 버퍼 메커니즘을 썼나요?
  * **A (키워드):** 메모리 버퍼 대신 File Buffer 영속성 확보, Chunk size/Flush interval 튜닝, 재시도(Exponential Backoff) 및 Dead Letter 로깅.
* **Depth 2:** 수집된 Kafka 쪽에서 특정 Consumer가 죽거나 파티션 처리가 늦어지는 지연(Lag) 현상은 어떻게 감지 및 방어했습니까?
  * **A (키워드):** Burrow/CloudWatch 활용Lag 오프셋 모니터링, 알럿 시스템. Consumer Group의 인스턴스를 파티션 수(Upper Bound)에 맞춰 선제적 Scale-out.
* **Depth 3:** 특정 뉴스나 포털 콘텐츠로 트래픽이 몰리면 Kafka 파티션(Partition) 레벨에서 핫스팟(Hotspot) 현상이 날텐데 프로듀서 측면에서 어떤 파티셔닝 전략을 구사했나요?
  * **A (키워드):** 단순 키 해시가 아닌, 라운드로빈(Round-Robin) 혼합 분산 처리, 특정 메가 컨텐츠 키에 Salting 무작위 난수 부여 처리.
* **Depth 4:** Consumer가 처리 중 GC 이슈 등으로 Rebalancing이 지속 발생하여 전체 파이프라인이 멈추는(Stop-the-world) 현상을 겪어봤나요? 어떻게 대처하나요?
  * **A (키워드):** `max.poll.interval.ms` 시간 상향, `max.poll.records` 갯수 하향, Consumer 내부의 가져오기 스레드와 처리(Process) 스레드 완전 분리 설계.
* **Depth 5:** 결판을 내서 질문하자면, 장애 시나리오별로 데이터 유실 방지와 비즈니스 처리량(Throughput) 방어 사이의 트레이드오프는 어떻게 극복했습니까?
  * **A (키워드):** 비즈니스 목적별 별도 분리 설계. 결제/로깅 등 중요 로그는 ISR=2, acks=all 셋업으로 퍼포먼스보다 정합성 치중. 단순 클릭스트림 로그는 acks=1로 Throughput 보장 레이어링.

---

## 4️⃣ [ClickHouse/서빙] OLAP 분석 및 피처 서빙 플랫폼 고도화

**Q. 실무 조직 지원을 위해 제로베이스(0 to 1)에서 실시간 ClickHouse OLAP를 도입했는데, 아키텍처적으로 수많은 DB 중 무엇을 고민하다 선택했고 한계는 없었나요?**
* **Depth 1:** 다양한 OLAP 엔진 중 굳이 ClickHouse를 선택한 기술적인 비교 우위와, 비즈니스 측면에서 사내 조직에 보장해야 했던 요건은 무엇입니까?
  * **A (키워드):** 비 다차원 큐브(Pre-aggregation) 제약 감소. 높은 Columnar 압축률, Vectorized Execution, 타겟팅 및 실시간 지표를 '수 초 이내'에 확인하려는 현업 부서의 강한 레이턴시 SLA 만족 가능성.
* **Depth 2:** ClickHouse 특유의 Materialized View(MV)를 설계할 때, 원본 Update/Delete 작업 시 발생하는 성능 저하와 Lock 이슈는 어떻게 핸들링했나요?
  * **A (키워드):** ClickHouse는 전통적인 Update에 취약(Mutation 병목). ReplicatedReplacingMergeTree 엔진 활용 및 잦은 변경분 데이터 비동기 백그라운드 병합 처리.
* **Depth 3:** 수십 명의 마케터/현업들이 Superset 대시보드 새로고침을 동시에 누르면 ClickHouse 클러스터에 부하가 급증합니다. 리소스 최적화 경험이 있나요?
  * **A (키워드):** 쿼리 Quota(할당량) 및 동시성 Max thread 제한 튜닝, 무거운 Join은 Airflow 단에서 사전 Aggregation Table로 프리-계산(Pre-calculated), 대시보드 쿼리 캐시 적극 개입.
* **Depth 4:** 광고 플랫폼 등에서 사용하기 위한 '피처(Feature) 서빙' 목적으로 HBase와 분산 OLAP 등을 사용할 때 각각 어떻게 역할을 분담했나요?
  * **A (키워드):** 단건 유저의 빠른 응답(Random Access K-V)이 필요한 서빙은 HBase. 광범위한 트래픽 패턴이나 모수 파악 범용 집계 등은 ClickHouse 채택으로 트래픽 특성 분리.
* **Depth 5:** 데이터 마트 모니터링 고도화와 알림 체계를 도입하고 난 후, 실제 비즈니스 의사결정 방식이 어떻게 달라졌습니까?
  * **A (키워드):** 데이터 지연 감소 및 이상치 사전 탐지. 마케터가 데이터 엔지니어에게 쿼리 요청하는 시간을 제거하여 캠페인 론칭 주기 50% 단축. 타겟팅 전환율(CVR) 직접적 상승 임팩트 달성.
