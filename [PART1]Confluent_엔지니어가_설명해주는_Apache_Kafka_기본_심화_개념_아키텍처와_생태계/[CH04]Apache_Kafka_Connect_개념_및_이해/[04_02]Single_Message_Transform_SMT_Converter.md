## [04_02] Single Message Transform(SMT), Converter

### Connect Worker
- Connector를 배포/구동시키는 프로세스
- Connector 배포 전에는
  - 단순히 Connect Worker Process가 떠있는 형태

### Connector 배포

#### Connector Class 지정
- Connector 배포를 위한 Configuration의 일부
  ```json
  "config": {
    ...
    "connector.class":"io.confluent...JdbcSinkConnector",
    ...
  }
  ```

#### Connect Worker에 Connector Instance 및 Task 생성
- Connector Task가 Source 시스템에서 데이터를 가져와서 Connect Record로 변환
- Connect Worker Process 내부
  - Connector
  - Task
    - Mysql에서 Native Data를 가져와서 Connect Record로 변환
- 대상 시스템을 연결하기 위한 Connector가 없다면,
  - Connect Framework를 기반으로 Custom Connector 개발 가능

### Converter 설정

#### Key, Value를 Byte Array로 변환하기 위한 Converter를 각각 설정
```conf
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://localhost:8081
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://localhost:8081
```
- schema.registry를 연동하게끔 되어 있음

#### Connect Record를 Byte Array로 변환
- 설정한 Converter가 Connect Record를 Byte Array로 변환한 후, Kafka로 전달
  - **Mysql** ---Native Data ---> **Task** --- Connect Record ---> **Converter**
- Converter도 필요시 커스텀 개발 가능

### Single Message Transform(SMT)
- 단건 메세지별 데이터 변환 기능
- Task와 Converter 사이에서 데이터 변환이 필요한 경우 사용
- Transform
  - `Cast`: 필드 또는 전체 Key,Value를 특정 유형으로 Cast
  - `Drop`: 레코드에서 Key, Value 제거 후 null로 설정
  - `InsertField`: 레코드 메타데이터 또는 구성된 Static Value의 속성을 사용하여 필드 삽입
  - `MaskField`: 필드 유형에 대해 유효한 null값으로 지정된 필드 마스킹
  - `ReplaceField`: 필드를 필터링하거나 이름 변경
- 약 20개정도의 `Pre-defied SMT` 제공

### SMT 설정
- Chaining 가능
  - **SMT를 여러개 연결**
  - `addTimeToTopic, labelBar`에 대해 설정
    ```json
    "config": {
      ...
      "transforms":"addTimeToTopic,labelBar",
      "transforms.addTimeToTopic.type":"org.apache.kafka.connect.trasforms.TimestampRouter",
      "tramsforms.addTimeToTopic.topic.format":"${topic}-${timestamp}",
      "tramsforms.labelBar.type":"org.apache.kafka.connect.transforms.ReplaceField$Value",
      ...
    }
    ```

### SMT
- 단건 메세지별 데이터 변환 가능
- Connect Record를 설정된 SMT 순서에 따라 변환
- **Mysql** --- `Native Data` ---> **Task** --- `Transform` --- `byte[]` ---> **Converter**
- SMT도 필요시 커스텀 개발 가능
- Connector 자체도 개발 가능

### Sink Connector의 Data Flow
- Source Connector의 역방향 순서
- Sink Connector의 Data Flow는 Source Connector의 역방향
- **Apache Kafka** --- `byte[]` ---> **Task** --- `Transform` --- `Native Data` ---> **Converter**

### Summary
- Single Message Transform(SMT)
- Converter