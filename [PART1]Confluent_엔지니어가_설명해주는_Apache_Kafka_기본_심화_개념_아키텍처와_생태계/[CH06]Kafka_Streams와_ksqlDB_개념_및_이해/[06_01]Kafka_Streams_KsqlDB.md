## [06_01] Kafka Streams, ksqlDB

### Realtime Event Stream Processing
- 실시간 이벤트 스트림 데이터 분석 및 처리
- 대용량의 이벤트 스트림 데이터를 실시간으로 분석 및 처리는 요구사항은 다양함

### 기존에 사용하던 Realtime Event Stream Processing 방법들
- Apache Spark, Storm, Flink

#### Apache Spark
- 범용적인 목적을 지닌 **분산 클러스터 컴퓨팅 프레임워크**
- MR 형태의 클러스터 컴퓨팅 패러다임의 한계를 극복하고자 등장
- **Spark Cluster** 구성 필요
- 이를 관리하는 Cluster Manager와 데이터를 분산 저장하는 Distributed Storage System 필요

#### Apache Storm
- Clojure 프로그래밍 언어로 작성된 **분산형 스트림 프로세싱 프레임워크**
- 별도의 **Storm Cluster**를 설치 구성
- **상태 관리가 지원되지 X**
  - Aggregation, Windows, Water Mark 등을 활용할 수 X

#### Apache Flink
- **통합 스트림 처리 및 Batch 처리 프레임워크**
- `Java/Scala`로 작성된 분산 스트리밍 Data Flow 엔진
- 사용자의 Stream Processing Code는
  - **Flink Cluster**에서 하나의 Job으로 배포 및 실행

### Kafka 진영에서 나온 Realtime Event Stream Processing 방법들

#### Kafka Streams
- Event Streaming용 Library(Java, Scala)
- **Framework이 아님 - 별도의 Cluster 구축이 불필요**
- `application.id`로 KStreams Application을 Grouping
- `groupBy, count, filter, join, aggregate`등 손수윈 스트림 프로세싱 API 제공

#### ksqlDB
- **Event Streaming Database**(또는 SQL Engine)
  - RDBMS/NoSQL DB가 X
- Confluent Community License(2017)
- 간단한 Cluster 구축 방법
  - 동일한 `ksql.service.id`로 `ksqlDB`를 여러개 구동
- 여러개의 Cluster는 `ksql.service.id`값을 서로 다르게 하기만 하면 됨
- SQL과 유사한 형태로 `ksqlDB`에 명령어를 전송하여 **스트림 프로세싱** 수행

### Kafka 기반 Event Stream Processing 방식
- 3가지 방식
  - Kafka Publish/Subscribe
    - Code를 길게 작성해야함
  - Kafka Streams
    - Kafka Pub/Sub보다 적은 코드량
  - ksqlDB
    - Kafka Streams보다 적은 코드량

### ksqlDB vs Kafka Streams
- SQL 개발 vs Java Application 개발
- Apache Kafka용 Event Stream Processing을 위한 방법
- ksqlDB
  - SQL을 사용하여 **실시간 이벤트 스트리밍 처리용 App**을 작성하기 위한
  - **Apache Kafka Streaming DB(SQL engine)**
- Kafka Streams
  - Java/Scala로 실시간 이벤트 스트리밍 처리용 App 및
  - micro service를 작성하기 위한
  - **Apache Kafka Streams Library**

### Kafka와의 상호작용 구조
- Broker와 별개로 구성
- ksqlDB
  - broker와 별개로 network로 read/write
- Kafka Streams
  - JVM application w/ Kafka Streams
  - network로 read/write
- 작업 FLOW
  - Topic으로 부터 데이터 Consume
  - 이후 1차 가공 토픽을 데이터 Produce

### 개발 방식 및 배포 방식
- SQL 배포 vs Application 배포
- **ksqlDB**
  - KSQL 쿼리 작성
  - 실시간으로 결과 확인
- **Kafka Streams**
  - Java/Scala로 Code 작성
  - 재컴파일 후 App 실행/테스트

### Summary
- Event Stream Processing
- Kafka Streams
- ksqlDB