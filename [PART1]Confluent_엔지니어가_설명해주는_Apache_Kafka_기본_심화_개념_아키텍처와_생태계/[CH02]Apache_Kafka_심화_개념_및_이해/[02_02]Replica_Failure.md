## [02_02] Replica Failure

### In-Sync Replicas 리스트 관리
- Leader가 관리함
- 메세지가 ISR 리스트의 모든 Replica에서 쉰되면 Commit된 것으로 간주
- Kafka cluster의 Controller가 모니터링 하는
  - Zookeeper ISR list에 대한 변경 사항은 Leader(Partition)가 유지
- `n`개의 replica가 있는 경우
  - `n-1`개의 장애를 허용할 수 있음

#### Follower가 실패하는 경우
- Leader에 의해 ISR 리스트에서 삭제
- Leader는 새로운 ISR을 사용하여 commit

#### Leader가 실패하는 경우
- Controller는 Follower 중에서 새로운 Leader 선출
- Controller는
  - 새 Leader와 ISR 정보를 먼저 Zookeeper에 Push한 다음
  - 로컬 캐싱을 위해 Broker에 Push

### ISR의 관리
- ISR은 리더가 관리
- Zookeeper에 ISR 업데이트
- Controller가 Zookeeper로부터 수신
- Follower가 너무 느리면 Leader는
  - ISR에서 Follower를 제거하고, Zookeeper에 ISR을 유지
  - `replica.lag.time.max.ms` 이내에 Follower가 fetch하지 않으면 ISR에서 제거
- Controller는 Partition metadata에 대한 변경사항에 대해
  - Zookeeper로부터 수신

### Leader Failutre
- Controller가 새로운 Leader를 선출하는 과정
- Controller가 새로 선출한 **Leader 및 ISR 정보**는
  - Controller 장애로부터 보호하기 위해 **Zookeeper**에 기록
  - 이후 client metadata update를 위해 모든 Broker에 전파
- FLOW
  - Broker 장애 발생
  - Controller가 `Zookeeper session timeout`을 통해 장애 감지
  - Controller가 새로운 Leader를 선출하고
    - 새로운 ISR 리스트를 Zookeeper에 기록
  - Controller가 모든 Broker에게 새로운 ISR을 Push
  - Client(C/P)는 metadata를 요청하여, 새로운 Leader 정보를 받음

### Broker Failure
- Broker 4대, Partition 4, Replication Factor 3일 경우 가정
- Partition 생성시 Broker들 사이에서 Partition들이 분산하여 배치
- Broker4에 장애가 발생한다면?
  - Partition Leader가 있던 Partition의 경우,  **Leader Failure**로직에 맞게 진행
  - Follower Partition만 존재할 경우, 그대로 운영
    - Leader 1개, Follower 1개

### Partition Leader가 없다면
- Leader가 선출될 때까지, 해당 Partition 사용 X
- `Producer::send`는 `retries` 파라미터가 설정되어 있으면 재시도
- `retries=0`라면, `NetworkException`이 발생
  - ack와 무관하게 아예 통신이 안되는 경우이기 때문

### 요약
- Follower가 실패하는 경우
  - Leader에 의해 ISR 리스트 삭제
  - Leader는 새로운 ISR을 사용하여 Commit
- Leader가 실패하는 경우
  - Controller는 Follower중 새로운 Leader 선출
  - Controller는 새 Leader와 ISR 정보를 먼저 Zookeeper Push
    - 이후 로컬 캐싱을 위해 Broker push
- Leader가 선출될 때까지 해당 Partition 사용 X