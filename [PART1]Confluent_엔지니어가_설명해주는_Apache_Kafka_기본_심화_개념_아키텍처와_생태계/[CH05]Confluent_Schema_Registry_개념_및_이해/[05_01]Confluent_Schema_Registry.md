## [05_01] Confluent Schema Registry

### Schema
- Data Structure
- 데이터를 만들어내는 Producer와 데이터를 사용하는 Consumer간의 계약으로 이용
- 스키마가 없으면
  - 시간이 지남에 따라, **제어된 방식**으로 데이터 구조를 발전시킬 수단이 X
  - 데이터 구조는 항상 **비즈니스**에 따라서 진화
    - 이를 `Schema Evolution`(스키마 진화)라고 함

### Schema Evolution
- 비즈니스가 변경되거나, 더 많은 어플리케이션이 동일한 데이터를 활용하기를 원함에 따라
  - 기존 **데이터 구조**가 진화할 필요성 발생

### AVRO
- Data Serialization System
- Apache Opensource Software Project
- **데이터 Serialization 제공**
- Java를 포함한 많은 프로그래밍 언어에서 지원
- 데이터 **구조 형식** 제공
- Avro 데이터는 **바이너리**이므로, 데이터를 효율적으로 저장

### AVRO의 장단점

#### 장점
- 압축, 고성능, Binary 포맷
- Java를 포함한 많은 프로그래밍 언어에서 지원
- Avro 데이터가 파일에 저장되면, **스키마가 함께 저장**
  - 나중에 모든 프로그램에서 파일 처리 가능
- **Avro 스키마**는 **JSON**으로 정의되므로,
  - 이미 JSON library가 있는 언어에서 구현 용이(`Schema Evolution`을 쉽게 지원)
- 데이터의 **타입**을 알 수 있음
- Confluent Schema Registry에서 사용 가능

#### 단점
- Binary 형태로 `Serialization`되기 때문에
  - 데이터를 쉽게 보고 해석하기 어려움
  - **디버깅, 개발시 불편**

#### Avro 형태
```json
{
    "namespace": "example.avro",
    "type":"record",
    "name":"User",
    "fields": [
        {"name": "name", "type":"string"},
        {"name":"favorite_number", "type":["int", "null"]},
        {"name":"favorite_color", "type":["string","null"]}
    ]
}
```

### Schema Evolution
- Compatibility
- e.g. Schema V1
  ```json
    {
        "namespace": "example.avro",
        "type":"record",
        "name":"User",
        "fields": [
            {"name": "name", "type":"string"},
            {"name":"age", "type":"int", "default":"-1"}
        ]
    }
  ```

#### Backward Compatibility(Default)
- 새로운 스키마를 사용하여 **이전 데이터를 읽는 것이 가능한가**
- 하위 호환성
- 필드 삭제 혹은 기본 값이 있는 필드 추가인 경우 가능
- e.g. Schema V2
  ```json
    {
        "namespace": "example.avro",
        "type":"record",
        "name":"User",
        "fields": [
            {"name": "name", "type":"string"},
            {"name":"age", "type":"int", "default":"-1"},
            {"name":"favorite_color","type":"string","default":"red"}
        ]
    }
  ```

#### Forward Compatibility
- 이전 스키마를 사용하여 **새로운 데이터를 읽는 것이 가능한가**
- 상위 호환성
- 필드 추가 혹은 기본값이 있는 필드 삭제
- e.g. Schema V2
  ```json
    {
        "namespace": "example.avro",
        "type":"record",
        "name":"User",
        "fields": [
            {"name": "name", "type":"string"}
        ]
    }
  ```

#### Full Compatibility
- **기본 값**이 있는 필드 추가 혹은 삭제
- e.g. Schema V2
  ```json
    {
        "namespace": "example.avro",
        "type":"record",
        "name":"User",
        "fields": [
            {"name": "name", "type":"string"},
            {"name":"favorite_color","type":"string","default":"red"}
        ]
    }
  ```

### Schema 설계시 고려할 점
- **삭제될 가능성**이 있는 필드라면 `default value`를 반드시 지정
- **추가되는 필드**라면 `default value`를 지정
- **필드의 이름 변경 X**
  - 수정이라는 개념이 없음

### Confluent Schema Registry
- 스키마 저장소
- 스키마의 중앙 집중식 관리 제공
  - 모든 스키마의 버전 기록 저장
  - Avro 스키마 저장 및 검색을 위한 **RESTful Interface** 제공
  - 스키마를 확인하고, 데이터가 스키마와 일치하지 않으면 예외 throw
  - 호환성 설정에 따라 Schema Evolution 가능
- 각 메세지와 함께 Avro Schema를 보내는것은 **비효율적**
  - 대신 Avro 스키마를 나타내는 `Global Unique ID`가 각 메세지와 함께 전송
- Schema Registry는 특별한 **Kafka Topic**에 스키마 정보 저장
  - `_schemas` topic
  - `kafkastore.topic` 파라미터로 변경 가능

### Schema 등록 및 Data Flow
- Avro type 사용, schema registry를 사용하는 상태
- Producer, Consumer 모두 Schema 본문은 Confluent Schema Registry를 참조
- Local Cache를 적극 활용

#### Producer FLOW
- Producer -> Confluent Schema Registry
  - 메세지 보내기 전에 Local Cache 확인
  - Schema가 없다면, Schema Send(Register) (w/ schema ID) to `Confluent Schema Registry`
- Confluent Schema Registry -> Kafka
  - Schema ID별로 Serialized된 Avro 데이터 Send

#### Consumer FLOW
- Consumer -> Confluent Schema Registry
  - Kafka로 부터 데이터를 가져옴
  - 가져온 데이터에 Avro Schema ID가 있다면
  - Schema ID별로 Deserialized된 AVRO 데이터 Read
- Local Cache에 없으면, ID로 Schema GET(from Confluent Schema Registry)

### Summary
- Schema
- Schema Evolution & Compatibility
- AVRO
- Confluent Schema Registry