## [02_03] Consumer Metric 이해

### Consumer 동작 방식 이해
- Consumer 내부에서 일어나는 일들
  - [Consumer Thread] `poll()` 호출
  - [Fetcher] Fetch 요청(to Broker)
  - [Broker] Fetch 결과 리턴
  - [Fetcher] Fetcher Queue에 Record가 담김
  - [Fetcher] Fetcher Queue에 담겨 있는 Record 반환(to Consumer Thread)

#### Consumer 내부 구간
- `poll` 함수 호출 평균 delay
- `poll`이 호출되지 않은 평균 시간

#### Consumer 전송 구간
- [Consumer/Topic/Partition 별]
- 초당 fetch 요청 건수
- 초당 메세지(record)/byte 건수
- Quota 제약으로 대기한 시간
- 평균 Commit 요청 처리 시간
- 초당 commit 요청 건수
- Partition 별 lag 개수

#### Coordinator 구간
- 초당 Heart beat 요청 건수
- 초당 Group join 건수
- 초당 Group Sync 건수
- Rebalance 평균 소요 시간
- 시간당 Rebalance 성공/실패 건수

### Consumer 내부 구간 이해
- Consumer 내부에서 처리하는 상태 확인
- `time-between-poll-avg`
  - Poll 함수를 다시 호출하기 까지 소요 시간
  - 모니터링
    - 값이 증가하면 `consumer group`에서 제외 가능
- `last-poll-seconds-ago`
  - 마지막 `poll()`을 호출한 이후 지난 시간
  - 모니터링
    - 값이 증가하면 consumer group에서 제외 가능
- `poll-idle-ratio-avg`
  - `poll`이 호출되지 않은 시간 비율(%)
  - 모니터링
    - 값이 증가하면 조치 필요
    - 가져오는 데이터를 줄이는 등

### Consumer 전송 구간 Metric 이해
- Consumer에서 Broker로 데이터 전송 및 수신 성능 확인
- `[Consumer] fetch-latency-avg`
  - 한번에 요청되는 레코드 건수
- `[Consumer] fetch-rate`
  - 초당 fetch 요청 건수
  - 모니터링
    - 증감 변화 확인
- `[Consumer] fetch-throttle-time-avg`
  - Quota를 초과하여 Broker 요청에 따라 대기한 시간
  - 모니터링
    - 값이 증가 시 quota값 확장
- `[Conumser,Topic] bytes-consumed-rate`
  - 초당 읽어온 byte 사이즈
- `[Consumer,Topic] fetch-size-avg`
  - 한번의 요청이 가져온 평균 byte 사이즈
  - 모니터링
    - 너무 작지 않은지 확인
- `[Consumer,Topic] records-consumed-rate`
  - 초당 가져온 레코드 평균 수
  - 모니터링
    - 이 값과 byte 사이즈도 함께 확인 필요
- `[Consumer,Topic] records-per-request-avg`
  - 한번에 가져온 평균 레코드 수
- `[Partition] records-lag`
  - partition의 가장 최근 lag값
  - 모니터링
    - 지속 커지지 않도록 관리
- `[Partition] records-lag-avg`
  - Partition의 평균 lag 값

### Coordinator 구간 Metric 이해
- Group Coordinator와 Consumer간 연결 상태
- `heartbeat-rate`
  - 초당 heartbeat 전송 건수
- `join-rate`
  - Partition rebalance를 위한 초당 Group Join 건수
  - 모니터링
    - 과도하게 많이 발생하는지 확인
- `sync-rate`
  - Partition rebalance를 위한 초당 Group Sync 건수
- `rebalance-latency-avg`
  - Partition rebalance 평균 소요 시간
  - 모니터링
    - Rebalance 소요 시간 확인
- `failed-rebalance-rate-per-hour`
  - 시간당 실패한 Partition rebalance 건수
- `commit-rate`
  - 초당 commit 요청 건수

### Consumer Rebalance 과정 이해
- Broker의 Group Coordinator와 Consumer간 진행 과정
- 새롭게 Consumer Group으로 참여한 consumer 들은
  - **Group Coordinator**를 통해 partition 할당

#### 신규 Consumer Group 생성 단계
- 동일한 Consumer Group의 Consumer가
  - Broker의 Group Coordinator에게 `JoinGroup` 요청 전송
- 가장 먼저 `JoinGroup` 요청을 보낸 Consumer가 Leader로 선정
- Leader는 Consumer Group에 partition 할당

#### Broker와 동기화 단계
- Partition을 할당 받은 consumer들은
  - Group Coordinator에 `SyncGroup` 요청 전송
- Broker의 Group Coordinator와 동기화가 완료되면,
  - 이후 데이터를 가져올 수 있음
