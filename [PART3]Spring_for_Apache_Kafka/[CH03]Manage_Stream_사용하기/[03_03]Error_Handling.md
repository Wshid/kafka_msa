## [03_01] Error Handling

### Retry Delivery
- RetryTemplate, Recovery Callback

### ErrorHandler
- SeekToCurrentHandler
  - DeadLetterPublishingRecoverer

### Retry Delivery 실습
```java
// In @Configuration
@Bean KafkaListenerContainerFactory,,,
ConcurrentKafkaListenerContainerFactory<String, Animal> factory = ...
factory.setConsumerFactory(animalConsumerFactory());
factory.setRetryTemplate(customizedRetryTemplate());

// retry time만큼 대기 이후 실행할 동작 정의
factory.setRecoveryCallback(context -> {
    ConsumerRecord record = (ConsumerRecord) context.getAttribute("record");
    System.out.println("Recovery callback. message=" + record.value());
    return Optional.empty();
    // ErrorHandler의 동작을 위한 exception 발생
    // throw new RuntimeException();
})
// container error가 발생하여야 동작
factory.getErrorHandler((thrownException, data) -> System.out.println("Error Handler. exception-" + thrownException.getMessage()))

private RetryTemplate customizedRetryTemplate() {
    return new RetryTemplateBuilder()
                .fixtedBackoff(1_000)
                .customPolicy(retryPolicy())
                .build();
}

private RetryPolicy retryPolicy() {
    Map<Class<? extends Throwable>, Boolean> exceptions = new HashMap<>();
    // 특정 에러가 발생하였을때의 처리 여부, true/false
    exceptions.put(ListenerExecutionFailedExecption.class, true);
    return new SimpleRetryPolicy(3, exceptions);
}
```

### SeekToCurrentHandler 실습
- DLQ를 활용할 예정
```java
// In @Configuration
@Bean KafkaListenerContainerFactory,,,
ConcurrentKafkaListenerContainerFactory<String, Animal> factory = ...
KafkaTemplate<String, Animal> kafkaJsonTemplate

factory.setConsumerFactory(animalConsumerFactory());
// Error가 발생하면, DeadLetterQueue로 전달
factory.setErrorHandler(new SeekToCurrentErrorHandler(new DeadLetterPublishingRecoverer(kafkaJsonTemplate)));
```
- 기존 토픽과 DLQ 토픽의 이름은 다음과 같음
  - topic: `clip3-animal`
  - DLQ: `clip3-animal.DLT`
