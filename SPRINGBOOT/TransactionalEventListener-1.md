> @TransactionalEventListener와 phase, condition에 대해 알아보자

## 예시 코드
```java
@RequiredArgsConstructor
@Service
public class SomeFacade {
    
    private final TestService testService;
    private final ApplicationEventPublisher applicationEventPublisher;
    
    @Transactional
    public void createTestEvent(String message) {
        log.info("""
                [log start]
                이벤트 - createTestEvent :: {}
                [log end]
                """, Thread.currentThread().getId()
        );

        //1. DB에 데이터를 넣는다.
        Test test = testService.createTestData(message);

        //2. 이벤트를 발생시킨다.
        applicationEventPublisher.publishEvent(
                TestEvent.builder()
                        .no(test.getNo())
                        .event(test.getName())
                        .build()
        );
    }
}


@RequiredArgsConstructor
@Component
public class EventComponent {
    
    private final KafkaProducer kafkaProducer;
    
    @TransactionalEventListener(
            phase = TransactionPhase.AFTER_COMMIT, //default
            condition = "#testEvent.no > 10"
    )
    public void publishTestEvent(TestEvent testEvent) {
        log.info("""
                [log start]
                이벤트 - publishTestEvent :: {}
                [log end]
                """, Thread.currentThread().getId()
        );
        
        // .. do something and
        
        kafkaProducer.sendTestEvent(testEvent);
    }
}
```

## TransactionalEventListener
- 상위 트랜잭션의 phase에 따라 호출여부의 영향을 받는 EventListener (= TransactionalApplicationListener)
- @Order 와 함께 사용해서 각 Listener의 우선순위를 설정할 수 있음
- 상위 트랜잭션이 없다면 실행되지 않지만, fallbackExecution=true로 지정하면 실행시켜줄 수 있다.

## Phase 종류
- TransactionPhase.AFTER_COMMIT
  - 말 그대로, 트랜잭션 커밋 이후에 이벤트 실행
    - 상위 트랜잭션의 커밋 이후이므로, 이벤트 내에서 @Transactional(propagation= REQUIRED, REQUIRES_NEW) 없이 호출되는 트랜잭션 이벤트는 전부 커밋되지 않는다.
    - @Async를 사용하면 다른 쓰레드이므로 다른 datasource를 사용하여 @Transactional 없이도 정상 실행
  - 상위 트랜잭션이 중간에 실패한다, 이벤트는 실행되지 않는다
- TransactionPhase.BEFORE_COMMIT
  - 트랜잭션 커밋 이전에 이벤트 실행
  - 다만 이벤트에서 exception이 발생하는 경우, 상위 트랜잭션에 영향을 미칠 수 있음 (별도로 확인 필요)
- TransactionPhase.AFTER_COMPLETION
  - 상위 트랜잭션의 성공, 실패 여부에 관계없이 항상 이벤트를 실행
- TransactionPhase.AFTER_ROLLBACK
  - 상위 트랜잭션이 정상 커밋되면, 이벤트가 실행되지 않는다
  - 상위 트랜잭션이 롤백되었을 때만, 이벤트가 실행된다

## Condition
- 특정 조건에만 이벤트를 처리하도록 하기 위한 값
- Object가 이벤트 실행 조건 파라미터로 넘어왔을 때, 해당 값을 참조 가능하다 (상위 코드에서 #testEvent 처럼)