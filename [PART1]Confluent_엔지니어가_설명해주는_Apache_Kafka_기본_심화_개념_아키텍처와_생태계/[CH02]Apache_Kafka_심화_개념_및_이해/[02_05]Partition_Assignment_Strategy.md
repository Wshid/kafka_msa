## [02_05] Partition Assignment Strategy

### Partition Assignment Strategy
- Range, RoundRobin, Sticky, CooperativeSticky, Custom
- Consumer의 설정 파라미터 중에서 `partition.assignment.strategy`로 할당 방식 조정
- `org.apache.kafka.clients.consumer.RangeAssignor`
  - Topic별로 작동
  - default
- `org.apache.kafka.clients.consumer.RoundRobinAssignor`
  - Consumer에게 Round Robin 방식으로 Partition 할당
- `org.apache.kafka.clients.consumer.StickyAssignor`
  - 최대한 많은 기존 Partition 할당을 유지하면서, 최대 균형을 이루는 할당 보장
- `org.apache.kafka.clients.consumer.CooperativeStickyAssignor`
  - 동일한 `StickyAssignor` 논리를 따르지만 협력적인 `Rebalance`를 허용
- `org.apache.kafka.clients.consumer.CustomPartitionAssignor`
  - 해당 인터페이스 구현시, 사용자 지정 할당 전략 사용 가능

### Range Assignor
- 가정
  - Topic 0, Topic 1, Partition 0, Partition 1, Consumer 0~2
- 분배
  ```md
  - T0-P0 --- consumer0
  - T0-P1 --- consumer1
  - T1-P0 --- consumer0
  - T1-P1 --- consumer1
  - consumer2는 idle state
  ```
- 특징
  - 순서대로 파티션 할당
  - 동일한 key를 가지고 있는 메세지들에 대한 topic간 `co-partitioning`하기 유리
    - delivery id를 key로 가지고 있는 `delivery_status`메세지와 `delivery_location` 메세지
  - Topic의 Partition 개수가 동일한 경우
    - `Co-Partitioning`이 가능함

### Round Robin Assignor
- Range 방식보다 **효율적**으로 분배하여 할당
- 가정
  - T0: P0 ~ P2
  - T1: P0 ~ P2
  - Consumer0,1
- 분배
  ```md
  T0-P0 --- consumer0
  T0-P1 --- consumer1
  T0-P2 --- consumer0
  T1-P0 --- consumer1
  T1-P1 --- consumer0
  T1-P2 --- consumer1
  ```
- 특징
  - `Reassign`후 Consumer가 동일한 Partition을 유지한다고 보장하지 X
  - **할당 불균형**이 발생할 가능성 존재
    - Consumer간 Subscribe 해오는 topic이 다를 경우, **할당 불균형**이 발생할 가능성 존재
    - e.g. C0~C2, T0~T2, T0:P0, T1:P0,P1, T2:P0~P2
      - C0-T0, C1-T0,T1, C2-T0,T1,T2 subscribe
      - 분배
        ```md
        T0-P0 --- consumer0
        T1-P0 --- consumer1
        T1-P1 --- consumer2
        T2-P0 --- consumer2
        T2-P1 --- consumer2
        T2-P2 --- consumer2
        ```
        - T2를 가져갈 대상이 C2밖에 없기 때문

### Sticky Assingor
- Range 방식보다 **Rabalncing Overhead**를 줄임
- 가능한한 균형적으로 할당 보장
  - Consumer들에게 할당된 Topic Partition수는 **최대 1만큼 다름**
  - Consumer A가 다른 Consumer B에 비해 **2개 이상** 더 적은 Topic Partition이 할당된 경우,
    - A에 할당된 Topic의 **나머지 Partition은 B에 할당 불가**
- 재할당이 발생했을 때, **기존 할당을 최대한 많이 보존**
  - Topic Partition이 하나의 Consumer에서 다른 Consumer로 이동할 때의 Overhead를 줄임
- 가정
  - C0~C2, T0~T3
  - T0~T3:P0,P1
  - C0,C1,C2 모두 T0~T3 subscribe
- 분배
  ```md
  T0-P0 --- consumer0
  T0-P1 --- consumer1
  T1-P0 --- consumer2
  T1-P1 --- consumer0
  T2-P0 --- consumer1
  T2-P1 --- consumer2
  T3-P0 --- consumer0
  T3-P1 --- consumer1
  ```
  - 만약 이 상황에서 C1이 제거된다면?
  - **기존 할당은 유지하면서, 나머지 부분 재할당**
- RR의 예외에서 분배
  - e.g. C0~C2, T0~T2, T0:P0, T1:P0,P1, T2:P0~P2
    - C0-T0, C1-T0,T1, C2-T0,T1,T2 subscribe
  - 분배
    ```md
    T0-P0 --- consumer0
    T1-P0 --- consumer1
    T1-P1 --- consumer1
    T2-P0 --- consumer2
    T2-P1 --- consumer2
    T2-P2 --- consumer2
    ```
    - RR에 비해, C2의 부하가 하나 줄어듦

### Summary
- `RangeAssignor`: default
- `RoundRobinAssignor`
- `StickyAssignor`