1. 선택 근거
왜 Flink 대신 Spark Streaming?

기존 EMR on EKS 스택 재사용, 운영 복잡도 최소화, 솔로 운영 환경, 현재 SLA(5초~1분)로 충분

Flink의 장점을 알면서도 선택한 게 기술 부채 아닌가?

의도적 tradeoff. 이벤트 단위 정합성/밀리초 레이턴시 요구 없음. 요구사항 변경 시 전환 로드맵 있음

Flink로 전환한다면 어떤 시점?

밀리초 레이턴시 요구, 복잡한 CEP, 대규모 stateful 연산, 이벤트 순서 정합성이 중요해지는 시점


2. 마이크로 배치 vs 진짜 스트리밍
Spark Streaming이 진짜 스트리밍이 아닌 이유?

이벤트를 하나씩 처리하지 않고 N초 단위로 묶어서 배치 처리. 레이턴시 하한선 존재

마이크로 배치의 레이턴시 하한선이 얼마나 돼?

실질적으로 100ms 이하 불가. 배치 스케줄링/태스크 기동 오버헤드 존재

Flink는 어떻게 진짜 스트리밍이 가능?

파이프라인 병렬처리. 이벤트 하나씩 operator 체인을 통과. 배치 경계 없음

Spark의 Continuous Processing 모드는?

실험적 기능으로 존재하나 production 사용 불가 수준. 제약 많음


3. 이벤트 타임 처리
이벤트 타임 vs 프로세싱 타임 차이?

이벤트 타임: 이벤트 실제 발생 시각. 프로세싱 타임: 시스템이 처리한 시각. 네트워크 지연으로 둘이 다를 수 있음

Spark에서 late data 처리?

watermark 설정으로 허용 지연 범위 지정. 범위 초과하면 drop

Flink의 이벤트 타임 처리가 더 정밀한 이유?

이벤트 단위로 워터마크 진행. Spark은 마이크로 배치 단위로 워터마크 갱신이라 정밀도 낮음

워터마크를 너무 크게 잡으면?

state 보존 시간 길어짐 → 메모리 압박. 결과 출력 지연

워터마크를 너무 작게 잡으면?

late data drop 증가 → 정합성 손실


4. Stateful 연산
Spark의 stateful 연산 종류?

mapGroupsWithState: 그룹당 정확히 1개 output
flatMapGroupsWithState: 그룹당 0개 이상 output
dropDuplicates: 중복 제거용 내장 state

mapGroupsWithState 실제 활용 예시?

세션 분석, 연속 실패 감지, 누적 지표, 중복 제거

state 크기가 너무 커지면?

메모리 압박 → GC 압박 → 성능 저하. TTL 설정으로 만료된 state 제거 필요

state TTL 설정 방법?

GroupStateTimeout.ProcessingTimeTimeout 또는 EventTimeTimeout으로 만료 처리

Flink의 state 관리가 더 나은 이유?

RocksDB state backend로 대규모 state를 disk로 overflow 가능. Spark은 메모리 의존도 높음

Flink RocksDB state backend란?

LSM Tree 기반 embedded DB. executor 로컬 disk에 state 저장. 메모리 초과 시 자동 spill. 체크포인트 시 S3 등에 snapshot


5. Exactly-once / 정합성
Spark Streaming의 exactly-once 보장 방법?

체크포인트 + idempotent sink 조합. Kafka offset 커밋과 sink write가 atomic하지 않으므로 sink가 멱등성 보장해야 함

Kafka offset 관리를 Spark이 직접 하나?

enable.auto.commit=false로 끄고 Spark이 체크포인트에 offset 저장. 직접 관리

장애 복구 시 offset이 rollback되면 중복 처리 아닌가?

맞음. 그래서 sink가 idempotent해야 exactly-once 의미론적 보장. Iceberg MERGE INTO가 그 역할

Flink의 exactly-once는 다른가?

Two-phase commit으로 Kafka offset 커밋과 sink write를 atomic하게 처리 가능. 진정한 exactly-once


6. 체크포인트
Spark Streaming 체크포인트에 뭐가 저장되나?

Kafka offset, state, 실행 계획 메타데이터

체크포인트 위치는?

S3 또는 HDFS. checkpointLocation 옵션으로 지정

체크포인트가 너무 자주 발생하면?

S3 write 오버헤드로 처리량 저하. 트리거 간격과 체크포인트 주기 조율 필요

코드 변경 후 체크포인트 호환성 문제?

실행 계획이 바뀌면 기존 체크포인트 사용 불가. 체크포인트 초기화 후 재시작 필요. state 유실 가능성


7. 성능 튜닝
Spark Streaming 처리량 높이는 방법?

트리거 간격 조정, 파티션 수 조정, executor 메모리/코어 튜닝, Kafka 파티션 수와 Spark 파티션 수 맞추기

Kafka 파티션 수와 Spark 파티션 수 관계?

Kafka 파티션 1개 = Spark 태스크 1개. Kafka 파티션이 병렬도의 상한선

backpressure 어떻게 처리?

maxOffsetsPerTrigger로 마이크로 배치당 처리할 최대 레코드 수 제한. 처리량 초과 시 Kafka에 쌓이게 두고 다음 배치에서 처리

Flink의 backpressure 처리와 차이?

Flink는 credit-based flow control로 자동 backpressure 조절. upstream operator가 downstream 처리 속도에 맞춰 자동 조절. Spark은 수동 설정 필요


8. 운영
Spark Streaming job 모니터링 어떻게?

Spark UI의 Streaming 탭 (배치 처리 시간, 입력 레이트, 처리 지연), CloudWatch 메트릭

처리 지연(lag) 감지 방법?

Kafka consumer group lag 모니터링. 배치 처리 시간이 트리거 간격 초과 시 알림

job 재시작 시 유의사항?

체크포인트 경로 유지. 코드 변경 있으면 체크포인트 호환성 확인. Kafka offset 연속성 확인

EMR on EKS에서 Streaming job 안정성?

executor pod 죽으면 Spark이 자동 재시도. driver pod 죽으면 job 자체 재시작 필요. driver HA 설정 권장