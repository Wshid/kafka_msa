## [02_09] Exactly Once Semantics EOS 2

### Transaction
- Transaction을 구현하기 위해, 몇 가지 새로운 개념들이 도입

#### Transaction Coordinator
- Consumer Group Coordinator와 비슷하게
- 각 Producer에는 `Transaction Coordinator`가 할당되며,
  - `PID` 할당 및 Transaction 관리의 모든 로직 수행

#### Transaciton Log
- 새로운 Internal Kafka Topic
- Consumer Offset Topic과 유사하게
- 모든 `Transaction`의 **영구적이고 복제된 Record**를 저장하는
  - Transaction Coordinator의 상태 저장소

#### TransactionalId
- Producer를 고유하게 식별하기 위해 사용
- 동일한 `TransactionalId`를 가진 `Producer`의 다른 인스턴스들은
  - 이전 인스턴스에 의해 만들어진 모든 `Transaction`을 재개/중단 할 수 있음

### Transaction 관련 파라미터

#### Broker Configs
- `transactional.id.timeout.ms`
  - **Transaction Coordinator**가 
  - Producer TransactionalId로부터 상태 업데이트를 수신하지 않고
  - 사전에 만료되기 전 대기하는 최대 시간
  - default: `604800000`(7d)
- **`max.transaction.timeout.ms`**
  - Transaction에 허용되는 최대 timeout
  - Client가 요청하는 Transaction 시간이, 이 시간을 초과하면
    - Broker는 `InitPidRequest`에서 `InvalidaTransactionTimeout` 오류 반환
  - Producer가 `Transaction`에 포함된 Topic에서
    - 읽는 `Consumer`를 지연시킬 수 있는 **너무 큰 시간 초과**를 방지
  - default: 900000(15m)
- `transaction.state.log.replication.factor`
  - Transaction State Topic의 Replication Factor
  - default: 3
- `transaction.state.log.num.partitions`
  - Transaction State Topic의 Partition 수
  - default: 50
- `transaction.state.log.min.isr`
  - Transaction State Topic의 min ISR 수
  - default: 2
- `transaction.state.log.segment.bytes`
  - Transaction State Topic의 Segment 크기
  - default: 104857600 bytes

#### Producer Configs
- `enable.idempotence`
  - 비활성화시, Transaction 기능 사용 불가
  - `true`이후, 다음 옵션을 같이 사용해야 함
    - `acks=all`
    - `retries>1`
    - `max.inflight.requests.per.connection=1`
  - default: false
- **`transaction.timeout.ms`**
  - Transaction Coordinator가 진행중인 Transaction을 사전에 중단하기 전에
    - Producer의 Transaction 상태 업데이트를 기다리는 최대 시간(ms)
  - 이 구성 값은 `InitPidRequest`와 함께 `Transaction Coordinator`에 전송
  - 이 값이 `> Broker::max.transaction.timout.ms`
    - `InvalidTransactionTimeout` 오류와 함께 요청 실패
  - default: 60000(60s)
- **`transactional.id`**
  - Transaction 전달에 사용할 `TransactionalId`
  - 이를 통해 Client는 새로운 `Transaction`을 시작하기 전에
    - 동일한 `TransactionalId`를 사용하는 새로운 `Transaction`이 완료되었음을 보장할 수 있으므로
    - 여러 Producer session에 걸쳐 있는 안정성 의미 체계 확보 가능
  - `TransactionalId`가 비어있으면(default), Producer는 `Idempotent Delivery`만으로 제한
  - `TransactionalId`가 구성된 경우, 반드시 `enable.idempotence`를 활성화해야 함
  - default: X

#### Consumer Configs
- `isolation.level`
  - `read_uncommitted`
    - offset 순서로 commit된 메세지와 commit되지 않은 메세지 모두 사용
  - **`read_commited`**
    - Non-transaction 메세지 또는 commit된 Transaction 메세지만 offset 순서로 사용
  - default: `read_uncommitted`
- `enable.auto.commit`
  - `false`: Consumer offset에 대한 Auto commit을 off
  - deafult: true
- Consume가 중복해서 데이터를 처리하는 것에 대해 **보장하지 않음**
  - Consumer의 중복처리는 **따로 로직을 작성해야 함**(Idempotent Consumer)
- e.g. 메세지를 성공적으로 사용한 후, Kafka Consumer를 이전 offset으로 되감으면
  - 해당 offset에서 최신 offset까지 모든 메세지를 다시 수신

### Transaction Data Flow 관련 예제 코드
- Consume 하고 Produce 하는 과정을 Transaction으로 처리
- https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging
- ![image](https://user-images.githubusercontent.com/10006290/206884383-9ad10c29-e1e3-4930-b2db-97c2c89f748e.png)
  - `initTransactions`로 시작
  - poll로 Source Topic에서 record를 가져옴
  - Transaction을 시작(`beginTransaction()`)
  - record로 비즈니스로직 수행 후, 결과 record를 target topic으로 send
  - `sendOffsetsToTransaction`을 호출하여
    - consume(poll)한 source topic에
    - consumer offset을 `commit`
  - `commitTransactiom/abortTransaction`으로 Transaction `commit/rollback` 수행

### Transaction Data FLOW
![image](https://user-images.githubusercontent.com/10006290/206884378-71f17bd4-ac54-4f59-9364-fb5c1260ed0d.png)
- 박스별 별개 머신을 나타냄(e.g. Consumer, Broker)
- Consume 하고 Produce 하는 과정을 Transaction으로 처리하는 과정
  - 그에 따른 Consumer offset 과정도 포함
- Topic의 Partition을 나타냄(하단)
- FLOW
  - 1) Transactions Coordinator 찾기
    - Producer가 `initTransactions()`를 호출하여, Broker에게 `FindCoordinatorRequest`를 보내서
    - Transaction Coordinator 위치를 찾음
    - Trnasaction Coordinator는 `PID`를 담당
  - 2) Producer ID 얻기
    - Producer가 `Transaction Coordinator`에게 `InitPidRequest`를 보내서(`TransactionalId` 전달) Producer의 PID를 가져옴
    - PID의 `Epoch`를 높여, Producer의 이전 `Zombie instance`가 차단되고, Transaction을 진행할 수 없도록 함
    - 해당 PID에 대한 mapping이 `2a`단계에서 `Transaction Log`에 기록
  - 3) Transaction 시작
    - Producer가 `beginTransactions()`를 호출하여 새 `Transcation`의 시작을 알림
    - Producer는 `Transaction`이 시작되었음을 나타내는 **로컬 상태**를 기록
    - 첫 번째 `Record`가 전송될 때까지 `Transaction Coordinator`의 관점에서는 `Transaction`이 시작되지 X
  - 4.1) AddPartitionsToTxnRequest
    - Producer는 `Transaction`의 일부로 새 `TopicPartition`이 처음 기록될 때 이 요청을 `Transaction Coordinator`에게 보냄
    - 이 `TopicPartition`을 `Transaction`에 추가하면 `Transaction Coordinator`가 `4.1a` 단계에서 기록
    - `Transaction`에 추가된 첫 번째 `Partition`인 경우 `Transaction Coordinator`는 `Transaction Timer`도 시작
  - 4.2) ProduceRequest
    - Producer는 하나 이상의 `ProduceRequests`(Producer의 `send()`에서 시작됨)을 통해
    - User Topic Partitions에 메세지를 `write`
    - 이러한 요청에는 `4.2a`에 표시된 대로 `PID, Epoch, Sequence Number`가 포함
  - 4.3) AddOffsetCommitsToTxnRequest
    - Producer에는 Consume되거나 Produce되는 메세지를 Batch 처리할 수 있는 `sendOffsetsToTransaction()`가 있음
    - `sendOffsetsToTransaction`메서드는 `groupId`가 있는 `AddOffsetCommitsToTxnRequests`를 `Transaction Coordinator`에게 보냄
    - 여기서 `Transaction Coordinator`는 내부 `__consumer_offsets` Topic에서
      - 이 `Consumer Group`에 대한 `TopicPartition`을 추론
    - `Transaction Coordinator`는 `4.3a`단계에서 `Transaction Log`에 이 `Topic Partition`의 추가를 기록
  - 4.4) TxnOffsetCommitRequest
    - Producer는 `__consumer_offset` topic에서 offset을 유지하기 위해
    - `TxnOffsetCommitRequest`를 `Consumer Coordinator`에게 보냄
    - `Consumer Coordinator`는 전송되는 PID 및 `Producer Epoch`를 사용하여 Producer가 이 요청을 할 수 있는지(Zombie X) 확인
    - `Transaction`이 `commit`될 때까지 해당 offset은 외부에서 볼 수 X
  - 5.1) EndTxnTrequest
    - Producer는 Transaction을 완료하기 위해 `commitTransaction()/aboutTransaction()`을 호출
    - Producer는 `Commit/Abort` 여부를 나타내는 데이터가 함께
      - `Transaction Coordinator`에게 `EndTxnRequest`를 보냄
    - `Transaction Log`에 `PREPARE_COMMIT`또는 `PREPARE_ABORT` 메세지를 write
  - 5.2) WriteTxnMarkerRequest
    - Transaction Coordinator가 각 Transaction에 포함된 `TopicPartition`의 Leader에게 이 요청을 보냄
    - 이 요청을 받은 각 Broker는 `COMMIT`(PID)또는 `ABORT`(PID) 제어 메세지를 로그에 기록
    - `__consumer_offsets` topic에도 `Commit(|Abort)`가 기록
    - Consumer Coordinator는 `commit`의 경우 이러한 오프셋을 구체화 하거나
      - `abort`의 경우 무시해야 한다는 알림을 받음
  - 5.3) Writing the final commit or Abort Message
    - `Transaction Coordinator`는 `Transaction`이 완료되었음을 나타내는
      - 최종 `COMMITED`또는 `ABORTED`를 `Transaction Log`에 기록
    - 이 시점에서 `Transaction Log`에 있는 `Transaciton`과 관련된 대부분의 메세지 제거 가능
    - Timestamp와 함께 완료된 `Transaction`의 PID만 유지하면 되므로
      - 결국 Producer에 대한 `TransactionalId -> PID` 매핑 제거 가능

### Summary
- Transaction 관련 파라미터
- Transaction Data Flow