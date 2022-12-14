## [04_01] Apache Kafka Connect

### Stream 간 메세지 전송
- Application 개발?
  - System간 메세지 전송이 필요할때마다 개발?
  - Producer, Consumer App

### Kafka Connector를 사용한 System간 메세지 전송
- Application 개발 불필요
- 이미 잘 만들어진 **Kafka Connector**를 손쉽게 사용
- Source Connector, Sink Connector
- 제품 구매 비용이 발생할 수 있음

### Kafka Connect? Connector?

#### Kafka Connect
- Apache Kafka 안팎으로 데이터를 **스트리밍**하기 위한 Framework
  - Kafka Connect는 다른 데이터 시스템을 Kafka와 **통합하는 과정을 표준화**한 Framework
  - 통합을 위한 Connector 개발, 배포, 관리를 단순화
  - **Connectors**
    - Task를 관리하여 데이터 스트리밍을 조정하는 Plugin(jar), Java Class/Instance
  - **Task**
    - Kafka와 다른 시스템간의 데이터를 전송하는 방법의 구현체(Java Class/Instance)
  - **Workers**
    - Connector 및 Task를 실행하는 실행중인 Process
  - **Converters**
    - Connect와 데이터를 보내거나 받는 시스템간에 데이터를 변환하는데 사용되는 Components(Java Class)
    - e.g. JsonConverters
  - **Transforms**
    - Connector에 의해 생성되거나
    - Connector로 전송되는 각 메세지를 변경하는 간단한 Components(Java Class)
  - **Dead Letter Queue**
    - Connect에서 Connector 오류를 처리하는 방법

### Connector는 어디서 찾을 수 있나?
- Confluent Hub
- Source Connector vs Sink Connector
- Confluent 지원 vs Partner 지원
- 무료 vs 상용
- Confluent Platform 및 Confluent Cloud 사용 가능 여부

### Connect Architecture
- Worker process가 Connector, Task 등을 관리
- Connect Worker Node 상에서 Connect Worker Process가 동작
- Connect Worker Process가 Connector Instance, Task Instance를 관리
- Connect Worker Process와의 kafka 접속 방법
  - bootstrap server 정보를 사용하여 kafka cluster에 연결

### Standalone vs Distributed Workers
- Single Process vs Multi Process(Scalability & Automatic Fault Tolerance)
- Standalone Worker
  - 확장 불가, 내결함성 제공 불가
  - **Local Disk Write(offset)**
- Distributed Workers
  - 확장 가능, 내결함성 제공 가능
  - 일반적인 방식
  - **Connect Related Topic(offset)**
    - 별도 토픽

### Multiple Distributed Connect Clusters
- `group.id`로 Cluster간 구분
  - Connector Worker 파라미터중 `group.id`를 다르게
- Connect Cluster 여러개가 하나의 **Connect Related Topic(offset)** 관리 가능

### Summary
- Kafka Connect Framework
- Connector, Tasks, Workers, Converters, Transforms, DLQ
- Confluent HUB
- Connect Architecture
- Standalone vs Distributed Workers