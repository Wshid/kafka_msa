## [02_03] Replica Recovery

### Replica Recovery

#### `acks=all`의 중요성
- 예시
  - 3개의 Replica로 구성된 하나의 Partition
  - Producer가 4개의 메세지(`M1 ~ M4`)를 보낸 상태
  - X, Y, Z 노드
    - X: Leader
    - Y, Z: Follower
  - broker별 메세지 상태
    - X: M1 ~ M4
    - Y: M1 ~ M2, M3 commit x
    - Z: M1 ~ M2
  - 이 때 X가 장애 발생
    - M3, M4 Commit 전 X 장애
  - Y가 Leader로 선출(Controller에 의해)
    - **Leader Epoch: 0 -> 1**
  - Z는 M3 fetch
    - Y는 **High Water Mark** 진행
      - High Water Mark: 복제 완료된 메세지 시점
    - Z는 fetch를 다시 수행, **High Water Mark**를 수신하고 진행
  - M3는 Commit이 된적이 없으나, Y가 M3를 가진 이유?
    - 새로운 리더가 선출되면, 새로운 리더의 데이터를 제거하지 않고, 그대로 활용
  - X는 M3, M4에 대한 ack를 Producer에게 보내지 x
  - Producer는 재시도에 의해 M3, M4를 다시 보냄
  - `idempotence=false`, M3 중복 발생
    - Y: M1, M2, M3, `M3`, `M4`
  - Z는 `M3`와 `M4`를 fetch
  - fetch를 다시 수행하고 High Water Mark를 수신 후 진행
- **결국 중복이 발생할 수 있는 상황**
  - 유실은 발생하지 x

#### `acks=1`이라면
- Y, Z가 복제를 못했던 M4는 어떻게 될지?
- Leader X가 장애나기 전에
  - Producer는 M4에 대한 ack를 수신
  - Leader는 다 받았기 때문
  - 단, 복제는 되지 X
- Y는 Leader로 선출
  - Leader Epoch: 0 -> 1
- Z는 M3 fetch
- Y는 High Water Mark 진행
- Z는 Fetch 다시 수행, High Water Mark를 수신하고 다시 진행
- **Producer**는 M4를 재시도 하지 x
  - 이미 받았다고 판정하므로(ack=1)
  - **M4 유실**
- **at most once**
  - 데이터 유실 발생 가능성이 충분히 존재

#### `acks=all`, 장애가 발생했던 X 복구시
- X가 복구되면, Zookeeper에 연결
- X는 Controller로부터 metadata를 받음
- X는 Leader Y로부터 Leader Epoch를 fetch
- X는 Leadership이 변경된 시점부터 **Truncate**
  - M2부터 Leader Epoch 증가
  - X에서는 M2이후의 메세지 제거
- Message 상태
  - X: M1, M2, ~~M3, M4~~
  - Y: M1, M2, M3, `M3`, `M4`
  - Z: M1, M2, M3, `M3`, `M4`
- X는 Leader Y로부터 복제
  - X: M1, M2, M3, `M3`, `M4`
- 복제가 한번 일어나면 ISR 리스트 복귀

### Availability와 Durability
- 가용성 or 내구성?

#### Topic 파라미터: `unclean.leader.election.enable`
- ISR 리스트에 없는 Replica를 Leader로 선출할 것인지에 대한 옵션(default: false)
- `false`
  - ISR 리스트에 Replica가 하나도 없으면, Leader 선출을 하지 X -> **서비스 중단**
- `true`
  - ISR 리스트에 없는 Replica를 Leader 선출 -> **데이터 유실**

#### Topic 파라미터: `min.insync.replicas`
- 최소 요구되는 ISR 개수 옵션
- `default: 1`
  - 리더만 제대로 동작해도 운영 가능하다는 의미
- 보통 현업에서는 `2`로 두고 사용
- `ISR < min.insync.replicas`
  - Producer는 `NotEnoughReplicas` 예외 수신
- 보통 `acks=all`과 함께 사용할 때 강력한 보장
  - + `min.insync.replicas=2`
- `n`개의 replica가 있고, `min.insync.replicas=2`
  - `n-2`개의 장애 허용 가능
- 예시
  - `replica:3`, `min.insync.replicas=2`, `acks=all`
  - **1개의 follower가 복제 성공시(리더 포함 2이므로), acks 반환** 
  - 1개의 장애 허용 가능(`3-2=1`)

#### 상황별 옵션 선택

##### 데이터 유실이 없게 하려면
- Topic: `replica.factor > 2`(>=3)
- Producer: `acks=all`
- Topic: `min.insync.replicas > 1`(>=2)

##### 데이터 유실이 있더라도 가용성을 높게 하려면
- `unclean.leader.election.enable=true`

### 요약
- replication.factor
- acks
- min.insync.replicas
- unclean.leader.election.enable