> Outbox Pattern
> - 트랜잭션과 메세징 시스템 2가지를 같이 사용해야될 때, 고려해볼만 한 패턴

### outbox 패턴을 고려하게 된 원인

특정 서비스 내에서
1. @Transactional 호출을 통한 DB에 CUD 동작 처리
2. kafka, rabbitMQ 등의 외부 메시징 시스템을 통해 트랜잭션 처리 이후 이벤트 발송

의 2가지 동작이 일어날 때  
- 1번 동작의 결과가 2번 동작에 영향을 미친다면 outbox pattern을 고려해볼만 하다.
- (ex. 1번 트랜잭션 커밋 시, kafka를 통해 메세지 publish 등)

1번과 2번은 아래의 이유로 같은 transaction에 묶는 것은 권장되지 않는다.  
- 네트워크 연결과 DB연결을 하나의 transaction으로 묶는 경우, 연결동안 connection pool을 계속 들고 있고
- 원하는 방식으로 동작하지 않는다(ex. 트랜잭션 실패에도 불구하고, 메세지 publish 되는 경우)

### outbox 패턴이란?
예를 들어, 1번 동작에서 메인으로 트랜잭션이 처리되는 DB table이 Order라고 가정해보자.  

실제 데이터가 처리될 Order 테이블 외에, Outbox라는 테이블을 추가로 두고  
Order 테이블에 데이터가 추가/삭제/수정 될 때, Outbox 테이블도 추가/삭제/수정 처리한다.

예시로, Outbox 테이블은 이벤트의 메세지나, current timestamp 값 등을 가질 수 있고  
이 테이블을 활용하여 외부 메세징 시스템에 이벤트 발행의 처리 여부를 관리한다.

```java
public void create(){
    transactionTemplate.executeWithoutResult(transactionStatus->{
        orderRepository.save(id,description);
        outboxRepository.save(new OutboxEntity(outboxId,message));
    });
    
    //이 프로세스 부터는 서비스 내무에 있을 수도 있고, 별도 외부 프로세스로 나눠져있을 수도 있다. 
    deliveryMessageQueueService.send(message);
    outboxRepository.delete(outboxId);
}
```

기본 동작은 아래와 같다.
1. Order 테이블과, Outbox 테이블의 트랜잭션은 같이 묶여야한다.
2. 외부 프로세스에서 Outbox 테이블에 조회 쿼리를 통해 질의하고, 데이터가 있으면 kafka 등 외부 메세징 시스템에 publish 한다.
3. 정상적으로 메세지가 publish됐다면, Outbox 테이블의 데이터를 제거한다.