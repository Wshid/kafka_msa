## [02_04] Consumer Rebalance

### Consumer의 동작 방식
- Partition에서 메세지를 Polling
- Consumer는 메세지를 가져오기 위해 `Partition`에 연속적으로 `Poll`
- 가져온 위치를 나타내는 `offset` 정보를 `__consuer_offsets` Topic에 저장하여 관리
  - e.g. `consumer group`별 `poll(100)`을 호출, 이후 어디까지 가져갔는지를 기록

### Consumer Load Balancing
- Consumer Group Id
- 동일한 `group.id`로 구성된 모든 Consumer들은 하나의 Consumer Group을 형성
- Consumer Group의 Consumer들은 작업량을 어느정도 균등하게 분할
  - partition key등 고려 필요
- 동일한 Topic에서 Consume하는 여러 Consumer Group이 있을 수 있음
- 다른 Consumer Group의 Consumer는 분리되어 독립적으로 작동

### Partition Assignment
- Partition을 Consumer에 Assign 할 때,
  - 하나의 Partition은 지정된 Consumer Group내의 하나의 Consumer만 사용
  - 동일한 Key를 가진 메세지는 동일한 Consumer가 사용
    - 단, Partition 수를 변경하지 않을 경우에 한함
  - Consumer의 설정 파라미터 중에서 `partition.assignment.strategy`로 할당 방식 조정
  - Consumer Group은 `Group Coordinator`라는 프로세스에 의해 관리
- **Group coordinator**(하나의 Broker)와 **Group Leader**(하나의 Consumer)가 상호 작용

#### FLOW

##### Consumer 등록 및 Group Coordinator 선택
- 각 Consumer는 `group.id`로 Kafka 클러스터에 자신을 등록
- Kafka는 Consumer Group을 만들고,
  - Consumer의 모든 offset은 `__consumer_offsets` Topic의 하나의 partition에 저장
- 이 Partition의 Leader Broker는 Consumer Group의 **Group Coordinator**로 선택

##### `hash(group.id) % offsets.topic.num.partitions` 수식을 사용하여
- `group.id`가 저장될 `__consumer_offsets`의 Partition을 결정
- e.g. `__consumer_offsets`의 파티션이 50개라면, 50보다 작은 수의 파티션에 offset 기록

##### JoinGroup 요청 순서에 따라 Consumer 나열
- Group Coordinator는 Group의 Consumer catalog를 생성하기 전에
  - Consumers의 `JoinGroup` 요청에 대해 `group.initial.rebalance.deplay.ms`(default: 3s)를 대기
    - e.g. 3초 안에 모든 Consumer가 JoinGroup 요청을 완료해야 함
- Consumer들이 `Consume`할 최대 `Partition`수까지 `JoinGroup` 요청을 수신하는 순서대로 Consumer 나열

##### Group Leader 결정 및 Partition 할당
- JoinGroup 요청을 보냈던 **최초 Consumer**를 Group Leader로 선정
- Group Leader는 구성된 `partition.assignment.strategy`를 사용하여 각 Consumer에게 Partition을 할당
- `Partition 수 > Consumer 수`, Consumer는 Consume할 Partition이 최대 1개만 존재
  - e.g. 마지막 할당 받은 Consumer는 할당받지 못할 수 있음
- `Partition 수 < Consumer 수`, Consumer는 Consume할 Partition이 n개 이상 존재

##### Consumer -> Partition mapping 정보를 **Group Coordinator**에게 전송
- **Group Leader**는 `Consumer -> Partition 매핑 정보`를 Group Coordinator에게 재전송
- Group Coordinator는 매핑정보를 **메모리에 캐시**하고 **Zookeeper에 유지**

##### 각 Consumer에게 할당된 Partition 정보를 보냄(by Group Coordinator)
- **Group Coordinator**는 각 Consumer에게 할당된 Partition 정보를 보냄
- 각 Consumer는 할당된 Partition에서 Consume 시작
- `Consumer 수 < Partition 수`: Partition이 할당되지 않은 Consumer는 **Idle**

### 왜 Group Cooridnator(a Broker)가 직접 Partition을 할당하지 않는가?
- Broker가 아닌 Consumer Leader에게 맡기는 이유
- **Kafka의 원칙**
  - 가능한 한 많은 계산을 **Client에서 수행**
  - Broker의 부담을 줄임
- 많은 Consumer Group과 Consumer들이 있고
  - Broker 혼자서 Rebalance를 위한 계산을 한다면
  - Broker에 엄청난 부담
  - 이러한 계산을 Broker가 아닌 **Client**에게 `Offload` 하는 것이 바람직

### Consumer Rebalancing Trigger

#### Rebalancing Trigger
- Consumer가 Consumer Group에서 **탈퇴**
  - **Group Coordinator**의 역할을 하는 Broker가 Consumer에게 주기적으로 **heartbeat**
  - 응답이 없을경우 죽은것으로 판단, rebalance trigger
- 신규 Consumer가 Consumer Group에 **합류**
- Consumer가 Topic subscribe를 **변경**
- Consumer Group은 Topic metadata의 **변경 사항**을 인지(e.g. partition 증가)

#### Rebalacing Process
- **Group Coordinator**는 `heartbeats` 플래그를 사용하여
  - Consumer에게 Reblance 신호를 보냄
- Consumer가 일시 중지하고, **offset commit**
- Consumer는 `Consumer Group`의 새로운 `Generation`에 합류
- Partition 재할당
- Consumer는 새 Partition에서 다시 Consume 시작

#### 유의점
- Consumer Rebalancing시 Consumer들은 메세지를 `Consume`하지 X
- 불필요한 Rebalancing은 **반드시 피해야함**

### Consumer Heartbeats
- Consumer 장애를 인지하기 위함
- Consumer는 `poll()`과 별도로 background thread에서 `heartbeats`를 보냄
  - `heartbeat.interval.ms`(default: 3s)
- 아래 시간동안 Heartbeats가 수신되지 않으면, Consumer는 `Consumer Group`에서 삭제
  - `session.timeout.ms`(default: 10s)
- `poll()`은 `heartbeats`와 상관없이 주기적으로 호출되어야 함
  - `max.poll.interval.ms`(default: 5m)

### 과도한 Rebalancing을 피하는 방법
- 성능 최적화에 필수

#### Consumer Group 멤버 고정
- Group의 각 Consumer에게 **고유한 `group.instance.id`**를 할당
  - id를 할당한다는 것은, 해당 토픽을 consume하는 파티션 고정
  - down되더라도, 나중에 해당 consumer가 복구될 것임을 가정함
- Consumer는 `LeaveGroupRequest`를 사용하지 X
- `Rejoin`은 알려진 `group.instance.id`에 대한 rebalance를 `trigger`하지 X

#### session.timeout.ms 튜닝
```bash
heartbeat.interval.ms = session.timeout.ms * 1/3

# session.timeoout.ms를 아래 값들의 사이값으로 설정
group.min.session.timeout.ms # default: 6s
group.max.session.timeout.ms # default: 5m
```
- 장점: Consumer가 **Rejoin**할 수 있는 더 많은 시간 제공
- 단점: Consumer가 **장애를 감지하는데 시간이 더 오래 걸림**

#### max.poll.interval.ms 튜닝
- Consumer에게 `poll()`한 데이터를 처리할 수 있는 충분한 시간 제공
- **너무 크면 안됨**

### Summary
- Consumer 파라미터인 `partition.assignment.strategy`로 partition 할당 방식 조정
- Consumer Rebalancing의 trigger 조건들
- 과도한 Consumer Rebalancing을 피해야 함