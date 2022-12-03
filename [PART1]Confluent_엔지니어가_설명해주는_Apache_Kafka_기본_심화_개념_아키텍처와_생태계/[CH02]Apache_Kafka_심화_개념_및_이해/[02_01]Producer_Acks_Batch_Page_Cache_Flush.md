## [02_01] Producer Acks, Batch, Page Cache, Flush

### Apache Kafka 주요 요소
- Producer, Consumer, Consumer Group

### Producer Acks

#### acks
- 요청이 성공할 때를 정의하는데 사용되는 Producer에 설정하는 Parameter

#### acks=0
- ack가 필요하지 X
- 메세지 손실이 다소 있더라도 빠른 메세지 송신이 필요할 때

#### acks=1
- default
- Leader가 메세지를 수신하면 ack를 보냄
- (Topic) Leader가 Producer에 ack를 보낸 후 Follower가 복제하기 전에 Leader에 장애가 발생하면 메세지 손실
- **At least once** 보장 

#### acks=-1, all
- 메세지가 Leader가 **모든 Replica까지 commit되면 ack를 보냄**
- Leader를 잃어도 데이터가 살아남을 수 있도록 보장
- 대기 시간이 더 길고, 특정 실패 사례에서 반복되는 데이터 발생 가능성 존재
- **At least once** 보장

### Producer Retry
- 재전송을 위한 Parameter
- 네트워크 또는 시스템의 일시적인 오류를 보완하기 위해 모든 환경에서 중요

| Parameter           | 설명                                            | Default     |
|---------------------|-------------------------------------------------|-------------|
| retries             | 메세지를 send하기 위해 재시도하는 횟수          | MAX_INT     |
| retry.backoff.ms    | 재시도 사이에 추가되는 대기 시간                | 100         |
| request.timeout.ms  | Producer가 응답을 기다리는 최대 시간            | 30,000(30s) |
| delivery.timeout.ms | send() 후 성공 또는 실패를 보고하는 시간의 상한 | 120,000(2m) |

- 보통 `retries`를 조정하기 보다, `delivery.timeout.ms` 조정으로 재시도 동작 제어
  - 전체적인 시간을 조정하는 변수
- `acks=0`에서 `retry`는 무의미

### Producer Batch 처리
- 메세지를 모아서 한번에 전송
- Batch 처리는 **RPC**(Remote Procedure Call) 수를 줄여서
  - Broker가 처리하는 작업이 줄어들기 때문에, 더 나은 처리량을 제공

#### linger.ms
- default:0, 즉시 보냄
- 메세지가 함께 Batch처리될 때까지 대기 시간

#### batch.size
- default: 16KB
- 보내기 전 Batch의 최대 크기
- Batch 처리의 일반적인 설정은
  - `linger.ms=100 & batch.size=1000000`

### Producer Delivery Timeout
- `send()` 후 성공 또는 실패를 보고하는 상한의 시간

#### Producer가 생성한 Record를 `send()`할 때의 Life Cycle
- **send()**
  - `max.block.ms`
    - 메세지를 담기 위한 `buffer` 할당 대기 시간
- `delivery.timeout.ms`
  - **batch**: `linger.ms`
    - 메세지가 함께 batch 처리될 때까지 대기 시간
  - **await send**
    - 데이터를 전송하고 기다림
  - **retries**: `retry.backoff.ms`
    - retry 사이의 대기 시간
  - **In flight**: `request.timeout.ms`
    - request에 대한 응답 대기 시간
  - **retries**와 **In flight**를 반복

### Message Send 순서 보장
- `enable.idempotence`
- 진행 중(in-flight)인 여러 요청(request)을 재시도하면 순서가 변경될 수 있음
- FLOW
  - Multiple in-flight request를 전송
    - `max.in.flight.requests.per.connection=5`(default)
    - Batch0 ~ Batch4
  - Batch0가 실패했지만, Batch1은 성공하면,
    - Batch1이 Batch0보다 먼저 `Commit Log`에 추가되어 순서가 달라짐
  - `enable.idempotence`를 사용할 때
    - 하나의 `Batch`가 실패하면, 동일 Partition으로 들어오는 후속 Batch들도
    - `OutOfOrderSequenceException`과 함께 실패

### Page Cache와 Flush
- 메세지는 Partition에 기록됨
- Partition은 `Log Segment file`로 구성(default: 1GB마다 새로운 Segment 생성)
- 성능을 위해 Log Segment는 **OS Page Cache**에 기록됨
- 로그 파일에 저장된 메세지의 데이터 형식은
  - Broker가 Producer로부터 수신한 것,
  - 그리고 Consumer에게 보내는 것과 정확히 동일하므로 **Zero-Copy**가 가능
- FLOW
  - **Producer** --send--> **Broker Process** --write--> **OS Page Cache** --flush--> **Disk**
- **Zero Copy**
  - 데이터가 User Space(e.g. Heap Memory in Java)에 복사되지 않고,
  - **CPU 개입 없이 Page Cache - Network Buffer 사이에서 직접 전송**되는 것을 의미
  - 이를 통해 Broker Heap 메모리를 절약하고, 엄청난 처리량 제공
- Page Cache는 다음과 같은 경우 Disk Flush
  - Broker가 종료
  - OS background "**Flusher Thread**" 실행

### Flush 되기 전에 Broker 장애가 발생하면?
- 이를 대비하기 위해 Replication이 필요
- OS가 data를 disk로 Flush하기 전에 **Broker의 시스템에 장애**가 발생하면, **데이터 손실**
- Partition이 Replication되어 있다면,
  - Broker가 다시 온라인 상태가 될 때, 필요시 `Leader Replica`에서 데이터가 복구됨
- Replication이 없다면 **데이터는 영구 손실**

### Kafka 자체 Flush 정책
- Flush(`fsync`) 트리거 하는 옵션
  - 마지막 Flush 이후의 메세지 수(`log.flush.interval.messages`)
  - 마지막 Flush 이후의 시간(`log.flush.interval.ms`)
- Kafka는 OS의 bg flush 기능(e.g. `pdflush`)을 더 효율적으로 허용하는 것을 선호,
  - 이러한 설정은 **기본적으로 무한(기본적으로 `fsync` 비활성화)로 설정**
- 이러한 설정을 **기본값으로 유지하는 것을 권장**
- `*.log`설정을 보면
  - 디스크로 flush된 데이터와
  - 아직 Flush되지 않은 Page Cache(OS Buffer)에 있는 데이터가 모두 표시됨
- Flush된 항목과, Flush되지 않은 항목을 표시하는 Linux 도구(e.g. `vmtouch`)도 있음

### 요약
- Producer Acks: 0, 1, -1(all)
- Batch 처리를 위한 옵션: `linger.ms, batch.size`
- 메세지 순서를 보장하려면 `Producer`에서 `enable.idempotence=true` 설정
- 성능을 위해 `Log Segment`는 `OS Page Cache`에 기록됨