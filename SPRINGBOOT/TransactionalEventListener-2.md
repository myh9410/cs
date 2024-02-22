> @Transactional과 @TransactionalEventListener에서 propagation 범위를 확인해보자

[이전 @Transactional의 propagation 확인 문서](https://github.com/myh9410/cs/blob/main/SPRINGBOOT/Transaction_Propagation-2.md#%EC%9D%BC%EB%B0%98%EC%A0%81%EC%9D%B8-transactional%EC%97%90%EC%84%9C-%EB%A1%A4%EB%B0%B1%EC%B2%98%EB%A6%AC)  
동작 사이에 ApplicationEventPublisher를 통해 이벤트를 발행

```java
    //1. integrateFacade
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void createUserAndDefaultBoard(String userId) {
        userService.createUserData();
        applicationEventPublisher.publishEvent(TestEvent.builder().no(1L).event("test1").build());
        boardService.createWelcomeBoard(1L);
    }

    //2. UserService
    @Transactional(propagation = Propagation.REQUIRED)
    public User createUserData() {
        // User Entity를 통한 사용자 데이터 생성
    }

    //3. BoardService
    @Transactional(propagation = Propagation.NESTED)
    public Long createWelcomeBoard(Long userNo) {
        // Board Entity를 통한 최초 게시판 데이터 생성
    }
    
    //4. EventComponent
    @Transactional(propagation = Propagation.REQUIRED)
    @TransactionalEventListener(
        phase = TransactionPhase.BEFORE_COMMIT
    )
    public void publishTestEvent(TestEvent testEvent) {
        // EventListener 내에서 데이터 처리
    }
```

@TransactionalEventListener의 phase 별 실제 이벤트 호출 여부와   
3.BoardService에서 RuntimeException이 발생 시, 롤백의 범위가 어떻게 되는지 확인

### RuntimeException 없이 성공하는 케이스
> 1. TransactionPhase.AFTER_COMMIT 인 경우
> ```shell
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.IntegratedService.createUserAndDefaultBoard]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> [nio-8080-exec-2] c.s.mq.infrastructure.EventComponent     : event call
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> ```
> 기존 트랜잭션이 정상적으로 끝난 상태에서 호출되고 트랜잭션이 반영된다.  
> 결과 : 2,3번 DB에 정상 커밋, 커밋 완료 후 4번 커밋

> 2. TransactionPhase.BEFORE_COMMIT 인 경우
> ```shell
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.IntegratedService.createUserAndDefaultBoard]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> [nio-8080-exec-2] c.s.mq.infrastructure.EventComponent     : event call
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
>```
> 2,3,4번의 트랜잭션이 정상적으로 커밋


### BoardService에서 RuntimeException이 발생한 케이스
> 1. TransactionPhase.BEFORE_COMMIT 인 경우
> ```shell
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard] after exception: java.lang.RuntimeException
>```
> RuntimeException으로 인한 rollback이 발생해서 eventListener가 아예 호출되지 않음  
> 결과 : 2,3번은 롤백, 4번은 호출되지 않음

> 2. TransactionPhase.AFTER_ROLLBACK 인 경우
> ```shell
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard] after exception: java.lang.RuntimeException
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.IntegratedService.createUserAndDefaultBoard] after exception: java.lang.RuntimeException
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> [nio-8080-exec-2] c.s.mq.infrastructure.EventComponent     : event call
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> ```
> 기존 트랜잭션이 롤백된 상태에서 호출된 트랜잭션은 반영되지 않는다.  
> 결과 : 2,3번은 롤백, 4번은 커밋되지 않음

### ApplicationEventPublisher 내에서 RuntimeException이 발생한 경우
```java
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void createUserAndDefaultBoard(String userId) {
        userService.createUserData();
        boardService.createWelcomeBoard(1L);
        applicationEventPublisher.publishEvent(TestEvent.builder().no(1L).event("test1").build());
    }

    @Transactional(propagation = Propagation.REQUIRED)
    @TransactionalEventListener(
            phase = TransactionPhase.BEFORE_COMMIT
    )
    public void publishTestEvent(TestEvent testEvent) {
        log.info("event call");
        callbackRepository.save(Callback.builder().no(testEvent.getNo()).count(3L).build());

        throw new RuntimeException();
    }
```

> 1. TransactionPhase.BEFORE_COMMIT 인 경우
> ```shell
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.UserService.createUserData]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> [nio-8080-exec-1] c.s.mq.infrastructure.EventComponent     : event call
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent] after exception: java.lang.RuntimeException
> ```
> 기존 트랜잭션이 미반영된 상태에서, applicationEventPublisher에서 rollback하여 전체 롤백 처리

> 2. TransactionPhase.BEFORE_COMMIT 인데, @Async로 TransactionalEventListener가 호출되는 경우
> ```shell
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.UserService.createUserData]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [         task-1] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> [         task-1] c.s.mq.infrastructure.EventComponent     : event call
> [         task-1] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [         task-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [         task-1] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent] after exception: java.lang.RuntimeException
>```
> @Async로 인해, 앞선 service들에서의 thread와 event에서의 thread가 다르고, tx가 유지되지 않는다.
> 따라서, createUserData와 createWelcomeBoard는 정상 커밋되고, event는 RuntimeException으로 인해 rollback된다. 

> 3. TransactionPhase.AFTER_COMMIT 인 경우
> ```shell
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.UserService.createUserData]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.web.services.BoardService.createWelcomeBoard]
> [nio-8080-exec-2] c.s.mq.infrastructure.EventComponent     : event call
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Completing transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent] after exception: java.lang.RuntimeException
> [nio-8080-exec-2] o.s.t.i.TransactionInterceptor           : Getting transaction for [com.springboot.mq.infrastructure.EventComponent.publishTestEvent]
> ```
> 기존 트랜잭션이 이미 커밋된 상태에서 AFTER_COMMIT으로 rollback 시도 시, 기존 트랜잭션은 반영되고, rollback 처리된 부분만 미반영된다.