# 1. Spark (분산 처리 엔진 최적화 및 파이프라인 구축)

## 📌 나의 경험 (이력서 기반)
- Spark 기반의 복잡한 ETL 프로세스 구현 및 대규모 배치·실시간 파이프라인 구축
- Spark SQL 로직 리팩토링 및 쿼리 최적화를 통한 성능 개선 및 코드 복잡도 해소
- 메타데이터 관리 및 Spark 퍼포먼스 튜닝을 통한 파이프라인 안정성 및 클러스터 성능 향상
- 중복 제거 및 멱등성 처리 로직을 통한 데이터 정합성 유지

## 🎯 토스뱅크 요구 역량 (JD 매핑)
- "Spark 등의 분산처리 엔진 실행계획 분석, 튜닝, 최적화 경험이 풍부하신 분"
- 대용량 데이터 처리를 위한 데이터 파이프라인(수집/처리/분석) 개발

## 💡 핵심 어필 포인트
단순히 Spark 파이프라인을 구축했다는 것을 넘어 실행계획(Execution Plan)을 들여다보고 어떻게 자원 효율이나 처리 시간을 수치적으로 개선했는지가 핵심입니다. 특히 트래블월렛과 줌인터넷에서 진행한 `쿼리 최적화`와 `퍼포먼스 튜닝` 과정의 기술적 디테일을 강조해야 합니다.

---

## 📝 예상 면접 질문 및 답변 준비 가이드

**Q1. Spark 실행 계획(Execution Plan)을 분석하고 튜닝하여 쿼리 성능을 개선한 구체적인 사례를 설명해 주세요.**
* **가이드:** 단순한 파라미터 조절 외에, Logical/Physical Plan 확인 후 Broadcast Join 채택, Shuffle 파티션 수 조정(Tuning), 데이터 Skewness 해결 등의 실제 경험을 구체적인 '전후 수치'와 함께 준비하세요.

**Q2. Spark 파이프라인을 운영하며 데이터 정합성 유지를 위해 멱등성(Idempotency)과 중복 제거를 어떻게 구현하셨나요?**
* **가이드:** 장애나 재처리 시에도 같은 결과를 보장해야 하는 금융권/결제 데이터 특성상 아주 중요한 질문입니다. 체크포인트 관리, Iceberg의 ACID 트랜잭션, Unique/Primary Key 활용 등 본인만의 아키텍처 및 로직을 정리하세요.

**Q3. Spark 파이프라인 환경에서 마주했던 가장 크리티컬한 장애(예: OOM, 메모리 누수)와 이를 해결한 과정(트러블슈팅)에 대해 말씀해 주세요.**
* **가이드:** 토스뱅크 JD에서 매우 강조하는 '트러블슈팅 경험'입니다. 문제 원인 파악 방식, 소스레벨 디버깅 유무, 그리고 해결 방안(Executor 메모리 튜닝, GC 튜닝, 로직 리팩토링)을 논리적으로 서술하세요.

**Q4. 대규모 분산 환경에서 로그 데이터 처리에 Spark를 사용할 때, 타 분산 엔진(예: Flink)과 비교해 어떤 한계점이 있었고 이를 어떻게 극복했나요?**
* **가이드:** "다른 도구와의 비교 및 장단점"은 단골 질문입니다. Micro-batch 방식의 한계나 데이터 딜레이 처리 경험에 대해 답변할 수 있게 준비하세요.

---

1. 선택 근거
왜 Flink 대신 Spark Streaming?
- 운영 복잡도 최소화, 솔로 운영 환경, 현재 SLA(5초~1분)로 충분. 이벤트 단위 정합성/밀리초 레이턴시 요구 없음. 요구사항 변경 시 전환 로드맵 있음

Flink로 전환한다면 어떤 시점?
- 밀리초 레이턴시 요구, 복잡한 CEP, 대규모 stateful 연산, 이벤트 순서 정합성이 중요해지는 시점

Flink 대비 spark streaming의 장단점은?
- 장점 : 
  - **배치/스트리밍 통합**: 기존 방대한 Spark SQL, MLlib 등 생태계와 코드를 공유(재사용)하기 매우 좋습니다.
  - **안정적인 장애 복구**: 마이크로 배치 단위 처리라 장애 발생 시 해당 배치만 재처리하면 되므로 트러블슈팅 및 복구가 명확합니다.
  - **운영 편의성과 레퍼런스**: 생태계가 워낙 넓어 YARN, Kubernetes 등 기존 인프라 환경에서 운영하기 편하고 이슈 해결 레퍼런스가 압도적으로 많습니다.
- 단점 : 
  - **태생적 지연(Latency)**: Native Streaming인 Flink와 다르게 Micro-batch 방식이므로 아무리 줄여도 본질적으로 수백 밀리초 ~ 수 초의 지연이 발생합니다.
  - **이벤트/상태 제어 한계**: 개별 이벤트 단위가 아니라서 이벤트 타임 기준으로 정밀하게 워터마크나 상태를 처리(CEP, 늦은 데이터 처리 등)하는 기능이 Flink에 비해 세밀하지 못합니다.
  - **Exactly-once 보장 복잡도**: 엔진 내 자체 Two-Phase Commit을 제공하는 Flink와 달리, Spark는 대상(Sink) 시스템의 멱등성(Idempotency)에 크게 의존하여 정합성을 구현해야 합니다.

Flink를 사용해보진 않았지만 아는대로 말해봐라


2. 이벤트 타임 처리
이벤트 타임 vs 프로세싱 타임 차이?
- 이벤트 타임: 이벤트 실제 발생 시각. 
- 프로세싱 타임: 시스템이 처리한 시각. 네트워크 지연으로 둘이 다를 수 있음

Flink의 이벤트 타임 처리가 더 정밀한 이유?
- 이벤트 단위로 워터마크 진행. Spark은 마이크로 배치 단위로 워터마크 갱신이라 정밀도 낮음

워터마크를 너무 크게 잡으면?
- state 보존 시간 길어짐 → 메모리 압박. 결과 출력 지연

워터마크를 너무 작게 잡으면?
- late data drop 증가 → 정합성 손실


4. Stateful 연산
Spark의 stateful 연산 종류?
- dropDuplicates: 중복 제거용 내장 state

state 크기가 너무 커지면?
- 메모리 압박 → GC 압박 → 성능 저하. TTL 설정으로 만료된 state 제거 필요


5. Exactly-once / 정합성
Spark Streaming의 exactly-once 보장 방법?
- 체크포인트 + idempotent sink 조합. Kafka offset 커밋과 sink write가 atomic하지 않으므로 sink가 멱등성 보장해야 함

Kafka offset 관리를 Spark이 직접 하나?
- enable.auto.commit=false로 끄고 Spark이 체크포인트에 offset 저장. 직접 관리

장애 복구 시 offset이 rollback되면 중복 처리 아닌가?
- 맞음. 그래서 sink가 idempotent해야 exactly-once 의미론적 보장. Iceberg MERGE INTO가 그 역할

Flink의 exactly-once는 다른가?
- Two-phase commit으로 Kafka offset 커밋과 sink write를 atomic하게 처리 가능. 진정한 exactly-once


6. 체크포인트
Spark Streaming 체크포인트에 뭐가 저장되나?
- Kafka offset, state, 실행 계획 메타데이터

체크포인트 위치는?
- S3 또는 HDFS. checkpointLocation 옵션으로 지정

체크포인트가 너무 자주 발생하면?
- S3 write 오버헤드로 처리량 저하. 트리거 간격과 체크포인트 주기 조율 필요

코드 변경 후 체크포인트 호환성 문제?
- 실행 계획이 바뀌면 기존 체크포인트 사용 불가. 체크포인트 초기화 후 재시작 필요. state 유실 가능성


7. 성능 튜닝
Spark Streaming 처리량 높이는 방법?
- 트리거 간격 조정, 파티션 수 조정, executor 메모리/코어 튜닝, Kafka 파티션 수와 Spark 파티션 수 맞추기

Kafka 파티션 수와 Spark 파티션 수 관계?
- Kafka 파티션 1개 = Spark 태스크 1개. Kafka 파티션이 병렬도의 상한선

backpressure 어떻게 처리?


8. 운영
Spark Streaming job 모니터링 어떻게?
- Spark UI의 Streaming 탭 (배치 처리 시간, 입력 레이트, 처리 지연), CloudWatch 메트릭

처리 지연(lag) 감지 방법?
- Kafka consumer group lag 모니터링. 배치 처리 시간이 트리거 간격 초과 시 알림

job 재시작 시 유의사항?
- 체크포인트 경로 유지. 코드 변경 있으면 체크포인트 호환성 확인. Kafka offset 연속성 확인

EMR on EKS에서 Streaming job 안정성?
- executor pod 죽으면 Spark이 자동 재시도. driver pod 죽으면 job 자체 재시작 필요. driver HA 설정 권장

---



