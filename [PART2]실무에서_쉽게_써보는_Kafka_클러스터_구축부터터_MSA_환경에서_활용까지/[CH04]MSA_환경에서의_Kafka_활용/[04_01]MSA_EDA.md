## [04_01] MSA & EDA

### Monolithic과 Micro Service Architecture
- MSA
  - 서비스 단위로 구분하여 구성
  - 서비스마다 RESTful API로 구성

#### Monolithic
- 장점
  - 소규모 서비스에서 개발, 테스트, 운영(및 모니터링 용이)
- 단점
  - 대규모 서비스에서 조그마한 수정에도 전체 빌드가 필요(빌드시간 증가)
  - 수정과 무관한 모듈의 재배포 불가피, scale-out 불리
  - 장애 전파 가능성 존재

#### MSA
- 장점
  - 각 서비스 독립적으로 개발, 테스트, 배포, scale-out 가능
  - CI/CD 달성 가능
  - 서비스별로 각기 다른 언어/DB 사용 가능
    - 각 서비스별 api로 추상화
- 단점
  - 개발 복잡도와 통신 오버헤드 증가
  - 통합 테스트 불리, **트랜잭션 처리**

### Event Driven Architecture
- Event
  - 시스템의 내부/외부에서 유발된 **시스템 상태**의 중요한 변화 또는 의미있는 사건
- Event Driven Architecture
  - 분산 시스템에서 비동기 통신 방식으로 이벤트를 **발생/구독**하는 아키텍처

#### 동기/비동기 통신
- 동기 통신
  - (RESTful API를 비롯한) API를 통한 요청-응답 방식(peer to peer)
    - peer가 꼭 살아 있어야 함
    - peer가 받지 못하면 유실됨
- 비동기 통신
  - Event Channel(Message Broker, Kafka)를 통한 pub/sub 방식

#### EDM = Event Driven Architecture를 적용한 MicroService
- 비동기 통신 사용
  - 각 MicroService간 **느슨한 결합도**(Loosely Coupled) 유지 가능
- EDM에서 발생한 이벤트는 **이벤트 스토어**에 저장(이벤트 로그)
- **Transaction Management**: Retry, Rollback