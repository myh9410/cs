> @Transactional의 propagation 별 트랜잭션 롤백 관계 확인

## 일반적인 @Transactional에서 롤백처리

```java
//1. IntegrateFacade
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void createUserAndDefaultBoard() {
    userService.createUserData();
    boardService.createWelcomeBoard();
}

//2. UserService
@Transactional(propagation = Propagation.REQUIRED)
public User createUserData() {
    // ...
}

//3. BoardService
@Transactional(propagation = Propagation.REQUIRED)
public Long createWelcomeBoard() {
    // ...
        
    throw new RuntimeException();
}
```

@Transactional의 default propagation은 REQUIRED
1. Propagation.REQUIRED  
   2번에서 트랜잭션이 성공하고, 3번에서 RuntimeException이 발생하면  
   상위 트랜잭션 - 1. IntegrateFacade 하위에 속한 트랜잭션들은 Rollback 처리

2. Propagation.NESTED  
   만약 3번은 롤백하고, 2번은 커밋되도록하는 구성을 가져가고 싶다면,  
   위의 코드에서 3번의 Propagation만 NESTED로 변경된다면, 2번은 커밋되고 3번은 롤백될까? NO
3. Propagation.NESTED With Try-Catch  
   아래의 경우들에서는 의도대로 2번은 커밋되고, 3번은 롤백된다.
    ```java
        //상위 트랜잭션에서 try-catch로 잡거나
        @Transactional(propagation = Propagation.REQUIRES_NEW)
        public void createUserAndDefaultBoard() {
            userService.createUserData();
            try {
                boardService.createWelcomeBoard();   
            } catch (Exception ex) {
                // ...
            }
        }

        //하위 트랜잭션에서 try-catch로 잡고, 현재 transaction의 롤백 처리를 설정
        public Long createWelcomeBoard() {
            try {
                // ...
   
                throw new RuntimeException();            
            } catch (Exception ex) {
                TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
            }
        }
    ```

4. 상위 트랜잭션을 제거한다면? 원했던 것 처럼 2번은 커밋되고, 3번은 롤백된다.
    ```java
        //1. IntegrateFacade
        public void createUserAndDefaultBoard() {
        userService.createUserData();
        boardService.createWelcomeBoard();
        }
        
        //2. UserService
        @Transactional(propagation = Propagation.REQUIRED)
        public User createUserData() {
             // ...
        }
        
        //3. BoardService
        @Transactional(propagation = Propagation.NESTED)
        public Long createWelcomeBoard() {
             // ...
        
            throw new RuntimeException();
        }
    ```