> 영속성 컨텍스트와 엔티티 생명주기에 대해 알아보자

### 비영속
- 영속성 컨텍스트로 등록되지 않은 상태
  - 엔티티 생성만 한 경우 (Transient한 상태)
```java
public class UserService {
    @Transactional(propagation = Propagation.REQUIRED)
    public User createUserData() {
        User user = User.builder().id("myh9410")
                .name("myh")
                .password("1234")
                .build();

        // 아래 결과는 false
        // contains - Check if the instance is a managed entity instance belonging to the current persistence context.
        log.info("entityManager contains : " + entityManager.contains(user));

        return userRepository.save(
                user
        );
    }
}
```
```
entityManager contains - service : false
```

### 영속
- 영속성 컨텍스트 관리 받는 상태
  - entityManager.persist() 시, 영속성 컨텍스트 관리 대상으로 등록한다.
  - transaction.commit() 시, 영속성 컨텍스트 관리 대상 객체들이 DB에 반영된다.
    - commit 시점에 entityManager.flush()가 발생하여 변경 감지(Dirty Checking)가 가능하다.

### 준영속
- 영속성 컨텍스트에 저장되었다가 분리된 상태 - entityManager.clear(), enetityManager.detach(), entityManager.close()
- 아래 코드는 영속 상태의 entity를 entityManager.detach()를 통해 관리 대상에서 제외하여 준영속 상태로 만든 것
```java
public class UserService {
    @Transactional(readOnly = true)
    public UserInfo findUserByNo(Long no) {
        UserInfo userInfo = userRepository.getUserInfoByNo(no);
        User user = entityManager.find(User.class, userInfo.getNo());

        log.info("entityManager contains - service : " + entityManager.contains(user));

        entityManager.detach(user);

        log.info("entityManager contains after detach - service : " + entityManager.contains(user));

        return userInfo;
    }
}
```
```
entityManager contains - service : true
entityManager contains after detach - service : false
```

### 삭제
- 엔티티가 삭제되어 transaction.commit()으로 인한 entityManager.flush() 시 데이터 삭제되는 상태
- 영속성 컨텍스트에 의해 관리되는 상태에서 entityManager.remove()를 통해 삭제 대상으로 등록

