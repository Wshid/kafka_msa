## [02_07] Kafka Log File

### Topic, Partition, Segment
- Physical View
- Partition은 Broker들에 분산되며, 각 Partition은 Segment File들로 구성됨
  - 하나의 파티션은 여러개의 Segment로 이루어짐
- Segment File Rolling Strategy
  - `log.segment.bytes`(default: 1G)
  - `log.roll.hours`(default: 168 hours) 

### Kafka Log Segment File Directory
- 각 Broker의 `log.dirs` 파라미터로 정의
- `Kafka Log Segment File`은 `Data File`이라고 부르기도 함
```bash
log.dirs=/data/kafka/kafka-loga,/data/kafka/kafa-log,b,...
```
- 각 topic과 그 partition은 `log.dirs`아래에 하위 디렉터리로 구성됨
- `test_topic`의 `partition 0`의 경우 `/data/kafka/kafka-log-a/test_topic-0`의 디렉터리로 생성됨

### Partition 디렉토리 안의 LogFile들
- 파일명에 의미가 있음
- `test_topic`의 `Partition 0`디렉터리로 생성되는 파일 예시

```bash
0000123453.index
0000123453.timeindex
0000123453.log
0000723453.index
0000723453.timeindex
0000723453.log
leader-epoch-checkpoint
```
- `0000123453.*` 파일은
  - 0000123453 offset 부터 0000723452 offset까지의 메세지를 저장/관리
  - 0000723452=0000723453-1

### Partition 디렉터리에 생성되는 파일들의 타입
- 확장자 혹은 파일명으로 구분함
- Partition 디렉터리에 생성되는 FileTypes는 최소 4가지
  - Log Segment File - 메세지 & metadata 저장
    - .log
  - index File - 각 메세지의 offset을 Log Segment 파일의 Byte 위치 매핑
    - .index
  - Time-based index file - 각 메세지의 timestamp를 기반으로 메세지를 검색하는 데 사용
    - .timeindex
  - Leader Epoch Checkpoint File - Leader Epoch과 관련 Offset 정보를 저장
    - leader-epoch-checkpoint
- 특별한 Producer 파라미터 사용시 Partition 디렉터리에 생기는 File Type
  - Idempotent Producer 사용시 -> `.snapshot`
  - Transactional Parameter 사용시 -> `.txindex`

### Log Segment File으의 특징
- 첫번째로 저장되는 메세지의 Offset 파일명이 됨
- Partition은 하나 이상의 Segment 파일로 구성
  - Partition0 - Segment0 ~ Segment3(Active)
  - Segment 0: 00000.log
  - Segment 1: 00030.log
  - Segment 3: 00444.log(Active)
- Log Segment File의 파일명은 해당 Segment File에 저장된 첫번째 메세지의 Offset

### Log Segment File Rolling
- 아래의 파라미터 중 하나라도 해당되면 새로운 Segment File로 Rolling
  - `log.segment.bytes`(default: 1G)
  - `log.roll.ms`(default: 168h)
  - `log.index.size.max.bytes`(default: 10M)
- `__consumer_offset`(Offset topic)의 segment file rolling 파라미터는 별도
  - `offsets.topic.segment.bytes`(default: 100M)

### Checkpoint File
- 각 Broker에는 2개의 Checkpoint File이 존재함
  - `log.dirs` 디렉터리에 위치
  - `replication-offset-checkpoint`
    - 마지막으로 Commit된 메세지의 ID인 High Water Mark 시작시,
    - Follower가 이를 사용하여 Commit되지 않은 메세지를 Truncate
  - `recovery-point-offset-checkpoint`
    - 데이터가 디스크로 Flush된 시점
    - 복구중 Broker는 이 시점 이후의 메세지가 손실되었는지 여부 확인


### Summary
- Segment File이 생성되는 위치는
  - 각 Broker의 `server.properties` 파일 안에서 `log.dirs` 파라미터로 정의함
    - comma로 구분하여 여러 디렉터리 지정 가능
  - 새로운 Segment File로 Rolling하기 위한 여러 파라미터 존재