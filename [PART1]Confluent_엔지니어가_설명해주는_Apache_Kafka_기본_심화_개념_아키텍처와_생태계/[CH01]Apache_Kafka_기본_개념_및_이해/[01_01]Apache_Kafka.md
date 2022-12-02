## [01_01] Apache Kafka
- Confluent 김현수 상무

### Apache Kafka는 무엇인가
- Event Streaming Platform
- Real-time Event Streams - Kafka - Real-time Event Streams

### Event란?
- 비즈니스에서 일어나는 모든 일(데이터)를 의미
- 이벤트를 비즈니스에서 활용

### Event Stream은 무엇인가?
- 연속적인 많은 이벤트들의 흐름
- Event는 BigData의 특징을 가짐
  - 비즈니스 모든 영역에서 광범위하게 발생
  - 대용량의 데이터(Big Data) 발생
- **Event Stream은 연속적인 많은 이벤트들의 흐름을 의미**

### Apache Kafka의 탄생
- Linkedin에서 개발
- 하루 4.5조개의 이상의 이벤트 스트림 처리
- 하루 3,000억개 이상의 사용자 관련 이벤트 스트림 처리
- **기존의 MQ(Messaging Platfom)으로 처리 불가능**
- **이벤트 스트림 처리를 위해 개발**
- 2011년에 Apache Software Foundation에 기부되어 오픈소스화

### Apache Kafka의 특징
- 이벤트 스트림을 안전하게 전송 Publish & Subscribe
- 이벤트 스트림을 디스크에 저장 Write to Disk
- 이벤트 스트림을 분석 및 처리 Processing & Analysis

### Apache Kafka 이름의 기원
- Franz Kafka 라는 작가에서 따옴

### Apache Kafka의 등장
- 2012년 최상위 오픈소스 프로젝트로 정식 출범
- Fortune 100 기업 중 80% 이상 사용

### Confluent Inc.
- Kafka 창시자가 만든 회사
  - Jay Kreps
- 2014년 설립

### Apache Kakfa의 사용 사례
- Event(메세지/데이터)가 사용되는 모든 곳에 사용
- Messing System
- IOT 디바이스로부터 데이터 수집
- 애플리케이션에서 발생하는 로그 수집
- Realtime Event Stream Processing(Fraud Detection, 이상 감지 등)
- DB 동기화(MSA 기반의 분리된 DB간 동기화)
- 실시간 ETL
- Spark, Flink, Storm, Hadoop과 같은 빅데이터 기술에 활용

### 산업 분야별 Apache Kafka 사용 사례
- 교통
  - 운전자 - 탑승자 매치
  - 도착예상시간(ETA) 업데이트
  - 실시간 차량 진단
- 금융
  - 사기 감지, 중복 거래 감지
  - 거래, 위험 시스템
  - 모바일 어플리케이션 / 고객 경험
- 오락
  - 실시간 추천
  - 사기 감지
  - In-App 구매
- 온라인 마켓
  - 실시간 재고 정보
  - 대용량 주문의 안전한 처리

### Apache Kafka의 성능
- 저렴한 장비로 초당 2M write

### Apache Kafka vs RabbitMQ
- Apache Kafka가 더 성능이 좋음
- Peak Throughput
  - Kafka: 605MB/s
  - Pulsar: 305MB/s
  - RabbitMQ: 38MB/s
- P99 Latency (ms)
  - Kafka: 5ms (200MB/s)
  - Pulsar: 25ms (200MB/s)