## [02_01] Producer Metric 상세

### 구간별 어떤 정보를 확인할 것인가?

#### FLOW
- Producer Thread ---`데이터 전송` ---> Record Accumulator(Buffer) ---`전송할 데이터 추출`---> Sender Thread
- 이후 데이터 전송, 전송 결과 리턴

#### Buffer 구간
- 데이터 압축률
- I/O 대기 시간(CPU 대기)
- 가용 Buffer 사이즈
- Buffer 할당 대기 시간
- Buffer 할당 대기 thread 수

#### Sender 구간
- 연결된 connection 관련 정보
- 인증 실패한 connection 정보
- Buffer의 데이터 체크 건수
- 전송 Queue 대기 시간
- 전송 부하로 인한 대기 시간(throttling time)
- 전송 실패 건수
- 재전송 비율

#### 전송 구간
- Producer/Broker/Topic별
- 초당 평균 전송 건수
- 초당 전송 데이터 사이즈
- 평균 요청 시간
- 초당 평균 record 전송 건수
- 초당 평균 record 전송 사이즈

### Producer의 동작 방식 이해
- Topic의 Partition 단위로 `buffer`에 저장하고, `partition`의 `leader`가 있는 broker로 전송
  - Key에 따라 특정 partition으로 데이터 전달
- Topic > partition의 Queue에 Batch size단위로 데이터 저장(in Record Accumulator(Buffer))
- Thread(Sender Thread)로 연결된 Buffer가 소유한 partition의 데이터를
  - Buffer에서 가져옴

#### 전체 관점으로 producer의 내부 상태 및 전송 성능 모니터링
- 데이터 압축률
- Record Accumulator(Buffer)
  - Buffer 사이즈 및 사용률
  - I/O 대기로 인한 CPU 유휴시간
- Sender(Thread)
  - connection 및 인증 정보
- Producer -> Broker
  - 전송 성능(요청 빈도, 전송 크기)
  - 전송 소요 시간
  - 전송 실패 건수
  - 평균 배치 크기, 평균 Record 크기
  - 평균 전송 record 건수
  - 응답 빈도

### Buffer 구간 Metric 이해
- `compression-rate-avg`
  - 전송할 데이터의 압축 수준
  - 원본 데이터와 압축된 데이터의 비율
  - **해당 수치가 낮을 수록 압축률이 높음**
  - 모니터링
    - 압축률 변화 추이
    - 압축률이 갑자기 변한다면, **입력 데이터의 변화** 확인
- `waiting-threads`
  - 데이터를 저장할 Buffer 메모리를 할당 받지 못해 대기 중인 thread 개수
  - 모니터링
    - 수치가 증가하면 `Buffer`의 메모리 크기가 적절한지 검토 필요
- `buffer-available-bytes`
  - 사용 가능한 buffer memory 크기
  - 모니터링
    - 여유가 있는지 확인
- `bufferpool-wait-time`
  - Buffer Pool에 데이터를 추가하기 위해 대기한 시간의 비율(%)
  - 모니터링
    - 비율이 높으면 새로운 buffer 메모리 확장
- `io-wait-time-ns-avg`
  - CPU가 I/O 작업이 완료될 때까지 기다린(idle) 평균 시간
  - I/O가 처리되는 시간동안 cpu는 다른 작업을 하지 못하고 대기
  - 모니터링
    - 높으면 cpu를 효율적으로 사용하지 못하므로 **disk 성능이 높은 것으로 변경 고려**
- `record-queue-time-avg`
  - 전송되기 전에 데이터가 Buffer에서 대기한 평균 시간(ms)
  - 모니터링
    - `linger.ms`이내로 관측되면 정상

### Buffer Pool 관련 내부 구조
- Sender Thread -> Batch List
  - 데이터를 Accumulator에 저장 요청
  - 저장할 공간이 부족하면 `Buffer Pool`에 새로운 Batch 할당 요청
- Buffer Pool의 내용을 Batch List에서 한개씩 할당하여 사용하는 형태

#### Buffer Pool에 가용한 메모리가 없다면?
- `RecordAccumulator`은 이후 요청은 차단하고, 메모리가 확보될 때까지 대기
- `Sender Thread`는 역시 `Batch` 공간을 할당 받기 위하여 대기(`waiting-thread` 증가)
- `Bufferpool_wait_time`은 이렇게 `send thread`가 대기한 비율을 나타내는 지표

#### 왜 Buffer Pool에 가용 메모리가 부족할까?
- `Producer`가 전송할 데이터를 읽어오는 속도가 너무 빨라서,
  - buffer pool의 메모리가 확보되지 못한 경우에 발생
- 이로 인해 `send thread`가 계속 대기하게 되고, 결국 **전송 성능(처리량)**이 저하됨

### Sender Thread 구간 Metric 이해
- `connection-count`
  - 현재 Broker와 연결된 connection 개수
  - 모니터링
    - 연결 수의 증감을 변화 확인
- `failed-authentication-rate`
  - 인증을 적용한 경우, 초당 인증 실패 건수
  - 모니터링
    - 미 인증된 producer의 접근 시도(인증을 시도한 대상 파악)
- `select-rate`
  - Sender Thread가 Buffer에 broker로 전송할 데이터가 있는지 I/O 체크(`select`)한 횟수(/sec)
  - 모니터링
    - 값이 증가하면 전송 가능한 상태의 데이터가 `buffer`에 없음을 의미
- `record-queue-time-avg`
  - 전송할 데이터가 buffer에 대기한 평균 시간
  - 모니터링
    - 지속 증가하는지 확인
- `record-error-rate`
  - 초당 전송 실패한 record 건수
  - 모니터링
    - 가능한 0으로 유지
- `produce-throttle-time-avg`
  - 브로커에 의해 요청을 중지(대기)한 평균 시간
  - 모니터링
    - 값이 증가하면 **브로커에 부하가 크다**고 판단 가능
    - 브로커를 증설하거나 브로커의 `Quota`를 증가

### 전송 구간 Metric 이해
- Producer 처리 성능 -> Broker별 처리 성능 -> Topic별 처리 성능
- `[producer]batch-size-avg`
  - 요청시 partition별 평균 전송 크기(byte)
  - 모니터링
    - `batch.size`보다 크다면 `batch.size` 증가
- `[producer]record-send-rate, [topic]record-send-rate`
  - 초당 전송된 record 건수
  - 모니터링
    - 수치 증감 확인
- `[producer]record-size-avg`
  - 전송된 record의 평균 사이즈
  - 모니터링
    - 평균 사이즈 관측을 통해 batch.size 증가 결정
- `[producer,broker]outgoing-byte-rate, [topic]byte-rate`
  - 초당 전송된 데이터 사이즈
- `[producer,broker]request-rate`
  - 초당 요청 건수
- `[producer,broker]response-rate`
  - 초당 브로커에서 응답을 받은 건수
  - 모니터링
    - ack옵션에 따라 응답시간에 차이 발생
    - 급격하게 증가/감소하는지 확인
- `[producer,broker]request-latency-avg`
  - 요청 후에 응답을 받기까지 소요된 평균 시간
  - 모니터링
    - 메모리가 충분하다면 `batch.size`를 늘려서
    - 처리량을 높이면서 `latency`는 유지되는지 확인
- `[producer,broker]request-size-avg`
  - 요청시 전송되는 데이터 평균 사이즈
