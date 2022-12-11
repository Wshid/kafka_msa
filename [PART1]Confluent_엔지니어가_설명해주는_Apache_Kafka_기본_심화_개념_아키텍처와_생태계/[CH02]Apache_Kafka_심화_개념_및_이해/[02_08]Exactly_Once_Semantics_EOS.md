## [02_08] Exactly Once Semantics EOS

### Delivery Semantics

#### At-Most-Once Semantics(최대 한번)
- 확인 시간이 초과되거나, 오류가 반환될 때
- Producer가 재시도하지 않으면,
  - 메세지가 Topic에 기록되지 않아, Consumer에 전달 x
- 중복 가능성을 피하기 위해 때때로 메세지가 전달되지 않을 수 있음을 허용

#### At-Least-Once Semantics(최소 한번)
- Producer가 Kafka Broker로부터 ack를 수신하고
  - `acks=all`이면, 메세지가 Kafka Topic에 **최소 한번**작성되었음을 의미
- 그러나 ack가 시간 초과되거나, 오류를 수신하면
  - Kafka Topic에 기록되지 않았다라고 가정하고, 메세지 전송을 재시도할 수 있음
- Broker가 ack를 보내기 직전에 실패했지만
  - 메세지가 Kafka Topic에 성공적으로 기록된 후에 재시도를 수행하면
  - 메세지가 2번 기록되어, 최종 Consumer에게 두번 이상 전달되어 **중복작업**과 같은 잘못된 결과로 이어질 수 있음

#### Exactly-Once Semantics(정확히 한번)
- Producer가 메세지 전송을 다시 시도하더라도
  - 메세지가 최종 **Consumer**에게 정확히 한 번 전달됨
- 메세징 시스템 자체와 메세지를 생성하고 소비하는 app의 협력이 한드시 필요
- e.g. 메세지를 성공적으로 사용한 후 Kafka Consumer를 이전 offset으로 되감으면
  - 해당 offset에서 최신 offset가지 메세지를 재수신

### Exactly Once Semantics(EOS)의 필요성
- 중복 메세지로 인한 **중복 처리 방지**
- 데이터가 `정확히 한 번`처리되도록 보장해야 하는 **실시간 미션 크리티컬 스트리밍 App**
  - 클라이언트(**Idempotent Producer**)에서 생성되는 중복 메세지 방지
  - **Transcation** 기능을 사용하여
    - 하나의 트랜잭션내의 모든 메세지가 모두 write 되었는지
    - 또는 전혀 write되지 않았는지 확인(Atomic Message)
- Use Cases
  - 금융 거래 처리 - 송금, 카드 결제 등
  - 과금 정산을 위한 광고 조회수 추적
  - Billing 서비스간 메세지 전송
- `Apache Kafka 0.11.0`(Confluent Platform 3.3)이전의 솔루션은
  - `At-Least-Once`를 활용하고,
  - 최종 사용자 app이 Consume 후 중복 제거를 수행하는 것이었음

### Exactly Once Semantics
- Java Client에서만 지원(>=0.11.0)
  - Producer, Consumer
  - Kafka Connect
  - Kafka Streams API
  - Confluent REST Proxy
  - Confluent KsqlDB
- **Transaction Coordinator** 사용 필요
  - 특별한 Transaction Log를 관리하는 Broker Thread
  - 일련의 ID번호(Producer ID, Sequence Number, Transaction ID)를 할당하고
    - 클라이언트가 이 정보를 메세지 Header에 포함하여 고유하게 식별
  - Sequence Number는 Broker가 **중복된 메세지**를 skip할 수 있게 함

### Exactly Once Semantics 관련 파라미터
- Idempotent Producer, Transactions
- Idempotent Producer
  - Producer의 파라미터 중 `enable.idempotence=true` 설정
  - Producer가 retry하더라도, 메세지 중복 방지
  - 성능에 영향이 별로 x
- Transaction
  - 각 Producer에 **고유한 transactional.id** 설정
  - Producer를 `Transaction API`를 사용하여 개발
  - Consumer에서 `isolation.level=read_committed`로 설정
    - transaction을 commit 했을 때, 그 트랜잭션의 메세지를 읽을 수 있도록 할껀가에 대한 옵션
- Broker의 파라미터는 Transaction을 위한 default 값이 적용되어 있음
  - 필요시에만 수정 필요

### Idempotent Producer 메세지 전송 프로세스
- 각 Producer는 고유한 `Producer ID`를 사용하여 메세지 송신
- Producer에는 `enable.idempotence=true`가 설정됨
- FLOW
  - 메세지는 Sequence Number와 고유한 Producer ID를 가지고 있음
    ```md
    Headers
    pid: 300
    seq: 0
    Key: "A"
    Value: "B"
    ```
  - Broker는 메모리에 `map {Producer ID: Sequence Number}`를 저장함
    - 이 map은 `*.snapshot`파일로 저장됨
- FLOW: Broker가 ack를 못 보낸 경우
  - Producer는 ack를 받지 못했으므로, 동일한 메세지에 대한 `retries` 수행
    - `enable.idempotence=true`설정을 하지 않았다면, Broker의 메세지 중복 수신 불가피
  - Broker는 DUP 응답 리턴
    - Broker가 체크하여 메세지가 중복된 것을 확인
    - 메세지를 저장하지 않고, Producer에게 `DUP Response`를 리턴

### 요약
- Producer의 파라미터 중 `enable.idempotence=true`
- 각 Producer에 고유한 `transactional.id` 설정
- Consumer에서 `isolation.lebel=read_committed`로 설정
  - 메세지를 보내어 저장은 되었으나, commit이 안된 데이터
  - 이 데이터를 읽는 것을 막을 수 있음 