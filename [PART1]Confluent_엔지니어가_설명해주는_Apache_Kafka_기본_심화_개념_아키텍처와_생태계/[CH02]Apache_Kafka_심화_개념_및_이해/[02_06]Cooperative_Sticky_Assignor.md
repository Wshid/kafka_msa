## [02_06] Cooperative Sticky Assignor

### Consumer Rebalacing Process
- 시간 흐름에 따른 Consumer Rebalance 과정
  - Consumer들이 JoinGroup 요청을 Group Coordinator에 보내면서 리밸런싱 시작
  - JoinGroup의 응답이 Consumer에게 전송(Group Leader는 Consumer들 정보 수신)
  - 모든 구성원은 Broker에 `SyncGroup` 요청을 보냄
    - Group Leader는 각 Consumer의 Partition 할당을 계산해서 Group Coordinator에게 전송
  - Broker는 `SyncGroup`응답에서 각 Consumer별 Partition 할당을 보냄

### Eager Rebalancing 프로토콜
- 지금까지 사용되었던 방식
- 최대한 단순하게 유지하기 위해 만들어짐
  - 각 구성원은 JoinGroup 요청을 보내고, 재조정에 참여하기 전에 소유된 **모든 Partition을 취소**(Revoke)해야함
  - 안전한 방법이긴 하나, `Stop-the-World` 프로토콜은
    - 그룹의 구성원이 **재조정 기간동안 작업을 수행할 수 X**

### Incremental Cooperative Rebalancing Protocol
- 이전 Eager Rebalancing 프로토콜보다 발전한 방식
- Revoke할 Partition만 Revoke하기
  - 할당에 있어서 변함이 없는 것은 Revoke없이 유지
- 이상적인 Consumer Rebalancing 프로토콜
  - 전체 재조정 동안 모두 정지 상태로 있는 대신에,
  - Consumer A만 하나의 Partition을 취소하는 동안만 가동 중지

#### 예시
- Consumer A,B가 Consume, Consumer C 추가
  - A: 1,3
  - B: 2
  - C: X
- 3번만 Revoke, A는 1번은 계속 Consume
- 2번도 계속 Consume
- 3번만 Consume되지 않다가, C에서 Consume

#### 이슈
- Consumer는 자신의 Partition 중
  - 어느것이 다른곳으로 **재할당**되어야 하는지 알 수 X

### Cooperative Sticky Assignor
- Rebalancing을 두번 수행
- JoinGroup 요청을 보내면서 시작하나
  - 소유한 모든 Partition을 보유
  - 그 정보를 Group Coordinator에 보냄
- Group Leader는 원하는 대로 Consumer에 파티션 할당
  - **소유권을 이전하는 Partition만 취소**
  - 3번 파티션만 revoke
- Partition을 취소한 구성원은 그룹에 ReJoin하여
  - 취소된 Partition을 할당할 수 있도록 두번째 **재조정**을 트리거
  - 재조정 요청 이후, Partition 3이 C에 할당
- Consumer A,B가 Consume, Consumer C 추가
  - A: 1,3
  - B: 2
  - C: X
- 3번 파티션만(재조정 대상만) **Synchronization Barrier**가 됨
- 두개의 balance 기간을 아래와 같이 정의
  - **1st rebanace**
    - Consumer는 자신의 Partition 중, 어느 것이 다른곳으로 재할당되어야 하는지 알게됨
    - 이후 revoke
  - **2nd rebanace**

### Cooperative Sticky Assignor History
- Basic Cooperative Rebalancing 프로토콜은 **Apache Kafka 2.4**에서 도입
- Incremental Cooperative Rebalancing 프로토콜은 **Apache Kafka 2.5**에서 추가
- **빈번하게 Rebalancing 되는 상황**이거나
  - **Scale-in/out으로 인한 downtime**이 우려가 된다면,
  - 최신 버전의 Kafka(>=2.5)기반으로 사용하는 것을 권장

### Summary
- Cooperative Sticky Assignor
- Incremental Cooperative Rebalancing 프로토콜은 **Apache Kafka 2.5**에서 추가