## [02_02] Kafka Broker Metric 상세

### Broker 구간별 모니터링
- Zookeeper와 기본적으로 통신

#### 파티션 관리
- 복제되지 못한 partition 개수
- Leader가 없는 partition 개수
- 접근 불가능한 Log 폴더 개수

#### Broker 내부 구간
- 요청 처리 시간
- 메시지 변환 시간/건수
- 대기 중인 producer 요청 건수
- 격리된 요청 건수
- 전체 Consumer Group 수
- Disk에 쓰는 시간

#### Broker Cluster 구간
- Active Controller 개수
- Controller Events 대기 시간
- Leader 선출 요소 시간
- Unclean Leader 선출 건수
- Broker -> Zookeeper 요청
- 처리시간

#### 전송 구간
- Producer, Consumer, Broker(follower)
- Connection 정보
- 메타정보 로딩 시간
- 초당 요청 건수 / 실패 건수
- 초당 record 전송 건수
- 초당 전송 데이터 크기
- 초당 에러 응답 건수

### 전송 구간 Metric 이해
- Broker 외부로 데이터 전송 및 수신 성능 확인
- `connection-count`
  - 현재 Broker와 연결된 connection 개수
  - 모니터링
    - 연결 수의 증감 변화 확인
- `ExpiredConnectionsKilledCount`
  - 재인증 실패로 종료된 connection 개수
  - 모니터링
    - 미인증된 접근 시도(인증 실패한 대상 파악)
- `partition-load-time-avg`
  - 메타 정보(consumer group, partition 등) 업데이트에 소요된 평균 시간
- `BytesInPerSec/BytesOutPerSec`
  - Broker에서 유입/유출된 초당 데이터 사이즈
  - 모니터링
    - Broker의 평균 처리량 증감 확인(얼마나 많이?)
- `RequestsPerSec/RequestBytes`
  - Broker에서 유입/유출된 초당 요청 건수
  - 모니터링
    - 얼마나 자주 요청하는지 증감 확인
- `MessagesInPerSec`
  - `Record(message)`의 초당 전송 건수
- `ErrorsPerSec`
  - 초당 에러 건수(response에 포함된)
  - 모니터링
    - 증가하면 원인 파악 필요

### Broker 내부구간 Metric의 이해
- `TotalTimeMs`
  - 요청을 처리하고 응답하는데 걸린 전체 시간(ms)
  - 모니터링
    - 평균 시간의 변화 확인
- `RequestQueueSize`
  - Broker에 전달된 요청이 Queue에 대기 중인 개수
  - 모니터링
    - 증가하면 원인 파악 필요(CPU, NW)
- `MessageConversionsPerSec`
  - Broker와 다른 Kafka version을 사용하는 경우 conversion 발생
  - 모니터링
    - 해당 client 버전을 broker 버전으로 변경 가이드
- `PurgatorySize`
  - Broker에 전달된 요청을 처리하지 않고 격리시킨 요청 건수
  - 특정 조건을 만족시키지 않는 경우
  - 모니터링
    - 수치 증감 확인(급증 시 원인 파악)
- `DelayQueueSize`
  - Producer의 요청 한도(quota)를 초과한 경우, 요청을 대기시킨 건수
  - 모니터링
    - 값이 지속 증가하면, Broker 증설 고려
- `ConsumerLag`
  - follower replica별 lag 사이즈
  - 모니터링
    - Lag값이 커지면 원인 파악
- `NumGroups`
  - 전체 Consumer Group 개수
  - 모니터링
    - 개수 변화 확인

### Total Time MS란?
- Broker에 전송된 요청을 처리하는데 소요된 **전체 시간**
  - 단계별 구간으로 구분
- Producer -> Request Queue Time -> Local Time -> Response Queue Time -> Response Time
- Local Time -> Remote Time(복제본 저장, Broker2)
- `RequestQueueTimeMs`
  - 요청큐에서 기다리는 시간
  - 높은 값은 `I/O Thread`가 부족하거나, CPU 부하 예상
- `LocalTimeMs`
  - 전달된 요청을 leader에서 처리하는 시간
  - 높은 값은 `disk I/O`가 낮음을 의미
- `RemoteTimeMs`
  - 요청이 Follower 작업 완료를 기다리는 시간
  - 높은 값은 `NW`연결이 늦어짐을 의미
- `ResponseQueueTimeMs`
  - 요청이 응답 큐에서 대기하는 시간
  - 높은 값은 `NW thread`가 부족함을 의미
- `ResponseSendTimeMs`
  - `Client`의 요청에 응답한 시간
  - 높은 값은 `NW Thread` 또는 `CPU`가 부족하거나, `NW` 부하가 높음을 의미

### Purgatory Size란?
- client의 요청이 **특정 조건**에 만족하지 않아,
  - 조건이 만족할 때까지 Broker 내부의 Queue에 보관된 건 수
- FLOW
  - Producer/Consumer ---`요청 조건 확인` ---> 처리 조건 확인 --- `미충족 요청 처리`---> Purgatory Queue ---`요건 충족시 요청 처리` ---> 요청 처리 ---`응답`---> Producer/Consumer
  - 위 FLOW는 Producer/Consumer외 Broker에서 처리

#### 어떤 조건에서 Purgatory에 격리될까?
- Producer
  - `request.required.acks=-1`인 경우
  - 데이터의 복제가 완료될 때까지 요청 격리
- Consumer
  - `fetch.min.bytes` 만큼의 데이터가 없는 경우
  - `fetch.wait.max.ms` 시간 만큼 요청 격리

#### Purgatory size가 커지면 어떤 조치를 해야할까?
- Producer의 경우
  - 복제 시간의 증가를 의미
  - `NW 대역폭` 및 Broker의 성능(Disk, Memory 등) 확인
- Consumer의 경우
  - 최소 `fetch size`가 너무 크거나
  - `Consumer`의 `poll`주기가 너무 빈번한지 검토 필요

### Kafka에서 Quota란?
- Client, User별로 요청 가능한 자원을 할당하여,
  - 특정 client가 전체 Clsuter에 영향을 주지 않도록 설정
- e.g. Producer
  - User=Alice의 요청
  - Alice의 최대 요청 크기 확인
  - 자원 사용률 확인
    - Client thread 최대 사용률 확인
    - Request Handler I/O Thread + Network Thread => Throttling Time 전달
  - Quota 초과시 Client에서 Throttling Time 전달
  - Client와 Connection 종료
  - [Consumer] Throttling Time 동안 대기. 요청 중지
- e.g. Consumer
  - Client.id에 맞게 요청 판단(Producer와 로직 동일)

#### 제한 가능한 Quota 유형
- Netowrk Bandwith 제한
  - `producerByteRate`: Producer가 전송 가능한 최대 크기
  - `consumerByteRate`: Consumer가 전송 가능한 최대 크기
- Request Rate 제한
  - `얼마나 자주 요청하는가`
  - 요청 client가 사용 가능한 최대 thread 사용률 제한
  - `Request Handler I/O Thread + Network Thread`
- Quota 적용 대상 및 방법
  - 적용 대상
    - **User**: 인증을 위해 생성한 User별 적용
    - **Client.id**: Producer 등에 지정한 고유 ID 값
  - Consumer의 경우
    - 최소 `fetch size`가 너무 크거나
    - `Consumer`의 `poll`주기가 너무 빈번한지 검증 필요

### Broker 내부 구간 Metric 2
- 자원 사용률 관련
- `RequestHandlerAvgIdlePercent`
  - Request Handler thread가 유휴 상태인 시간의 평균
  - 모니터링
    - 이 수치가 높으면 일을 안한다는 의미
- `NetworkProcessorAvgIdlePercent`
  - 네트워크 프로세서가 유휴 상태인 비율
  - 모니터링
    - 낮을수록 thread가 많은 작업을 하고 있음을 의미
- `LogFlushRateAndTimeMs`
  - Cache에 저장된 데이터를 Log(Disk)로 저장하는 비율
  - 모니터링
    - 증가한다면 Broker 추가 및 SSD 도입 고려

### Broker Cluster 구간 Metric 이해
- Kafka Cluster에서 발생하는 다양한 Event 및 운영 안정성 관련된 지표들
- `ActiveControllerCount`
  - Kafka Cluster에 존재하는 Active Controller 개수
  - 모니터링
    - 1개만 존재해야 함
- `EventQueueTimeMs`
  - Controller에 전달되는 event를 처리하기 전에 queue에 대기한 시간
- `LeaderElectionRateAndTimeMs`
  - Leader가 없는 partition이 발생한 경우, 새로운 leader를 선출하는데 소요된 시간
  - 모니터링
    - 증가하면, leader가 없는 상태가 오래 유지
- `UncleanLeaderElectionsPerSec`
  - 검증되지 않은(데이터 유실이 있을 수 있는 `OSR` 중에서) partition이 leader로 선출되는 초당 건수
  - 모니터링
    - Broker 설정을 `enable`해야 측정 가능
    - 데이터 유실이 있더라도 **가용성**을 높이기 위한 설정
    - 증가하면, 데이터 유실 발생을 의미

### Unclean Leader Election이란?
- 서비스 가용성을 위하여 **데이터 유실**이 있는 Partition을 leader로 선정하여, 서비스를 운영하는 방식
- **가용성**이 중요한 서비스

### Broker Partition 관련 이해
- Broker에서 관리하는 `partition`들의 정상 및 비정상 상태 관리
- `UnderReplicatedPartitions`
  - 전체 replica 개수보다 ISR 개수가 적은 partition 수
  - `Leader`에서 `Follower`로 복제가 늦어지는 경우 수치 증가
  - Broker가 장애일 때 수치 증가
  - 모니터링
    - **0 유지**
    - 정상 상태에서는 전체 `replication`수와 `ISR` 개수 동일
    - `ISR count < all replicas count`
- `UnderMinIsrPartitionCount`
  - 최소 replica 개수보다 ISR 개수가 적은 Partition 수
  - 모니터링
    - **0 유지**
    - `ISR count < min replicas count`
- `OfflinePartitionsCount`
  - `Leader partition`이 없어서 서비스(rw)가 불가능한 파티션 수
  - 오직 `leader partition`만 rw가 가능함
  - 모니터링
    - `>0`, 빠르게 조치 필요
- `OfflineReplicaCount`
  - `Follower partition` 중에서 연결이 안되는 `partition` 개수
  - 모니터링
    - `0` 유지
- `OfflineLogDirectoryCount`
  - HW 장애로 접근이 불가능한 log directory 개수
  - 모니터링
    - **0 유지**
    - 증가하면 장애로 판단하고, 즉시 대응
