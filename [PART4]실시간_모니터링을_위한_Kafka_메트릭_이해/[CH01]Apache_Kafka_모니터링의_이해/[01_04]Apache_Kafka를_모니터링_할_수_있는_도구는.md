## [01_04] Apache Kafka를 모니터링 할 수 있는 도구는?

### 유형별 모니터링 도구
- 직접 조회(console)
  - JMX Metric 직접 수집
  - jconsole, jmx 등
- Open source
  - 무료, OSS 장단점 존재
  - CMAK, AKHQ 등
- 상용 서비스 구매
  - 다양한 기능 제공
  - Confluent, DataDog 등
- Apache Kafka 모니터링 시스템 직접 구축
  - Elastic Stack 기반
  - Proemtheus + Grafana 기반

### 직접 조회 방식
- `Jconsole Tool`을 통해 Kafka Broker에 직접 접속하여 정보 조회(Broker별 수집)
- `JMX Tool`을 통해 Console에서 직접 Kafka Broker에 연결하여 Metric 조회
  ```bash
  >domain kafka.network # 수집할 metric domain 선택
  >bean kafka.netowrk:name=TotalTimeMs,request=Produce,type=RequestMetrics #TotalTimeMs 지표 선택
  >get Mean # 수집된 지표의 평균(Mean) 지표 출력
  ```

### OpenSource
- CMAK
  - Broker별로 사전에 정의된 metric을 조회 가능
    - 사용자가 metric을 추가/수정하기 어려움
  - Topic별로 사전에 정의된 metric 조회 가능
    - partition 재조정과 같은 운영기능 제공
- AKHQ
  - Topic별로 사전에 정의된 metric 조회 가능
    - live data 확인 가능
  - Kafka에 저장된 메세지 확인
  - Partition, Lag등 세부 지표 확인 가능

### 상용 서비스 구매
- DataDog
  - 서비스 가입 후, 각 Broker에 Agent를 설치하면 기본 모니터링 dashboard 제공
  - 구간별 중요 지표 제공

### 모니터링 시스템 구축
- Elastic Stack
  - 실시간 jmx metric 수집 및 시각화
  - Input Message/Out Message 성능 측정 가능

### 모니터링 유형별 특징
- 수집 방식에 따라서 운영 편의성, 장/단점, 기능 등이 다르며, 업무 목적에 따라 선택 필요

| 구분                      | 직접 조회                        | Open Source                  | 상용 서비스                                                      | 자체 모니터링 시스템                          |
|---------------------------|----------------------------------|------------------------------|------------------------------------------------------------------|-----------------------------------------------|
| **설치 & 구성**               | 단순                             | 약간 복잡                    | 가장 단순                                                        | 복잡                                          |
| **업무 활용도**               | 낮음                             | 높음                         | 매우 높음                                                        | 매우 높음                                     |
| **멀티 Kafka Cluster**        | 불가                             | 가능                         | 가능                                                             | 가능                                          |
| **JMX Metric 선택 및 시각화** | 가능                             | 불가                         | 가능                                                             | 가능                                          |
| **Topic 관리(생성, 삭제)**    | 불가                             | 가능(일부)                   | 불가                                                             | 불가(별도 기능 추가 가능)                     |
| **Partition 관리(변경)**      | 불가                             | 가능(일부)                   | 불가                                                             | 불가(별도 기능 추가 가능)                     |
| **장점**                      | 간단하게 metric 조회             | 쉽게 다양한 기능 활용        | 쉽게 효율적인 모니터링                                           | 업무에 최적화된 모니터링                      |
| **단점**                      | 다양한 지표의 직관적 파악 어려움 | 업무에 필요한 기능 추가 불가 | 서버당 라이센스 비용 부과, 외부망 연결 필요(Cloud 서비스의 경우) | 구축 및 운영 비용                             |
| **대표 도구**                 | Jconsole, jmx, jolokia           | CMAK, AKHQ, kafka-monitoring | Datadog, Confluent Control Center, Iense                         | Elastic stack 기반, Prometheus + Grafana 기반 |
