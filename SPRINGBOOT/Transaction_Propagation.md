> Transaction 과 그 propagation에 대해 알아보자.

- 스프링의 트랜잭션 기능에 대해 설명해주세요, 트랜잭션 동작원리
    - @EnableTransactinoManagement -> transaction manager가 설정됨 -> propagation 등에 따라 commit과 rollback 범위가 설정되고 manager를 통해 실행 후 commit이나 rollback 여부가 결정됨

- Transaction Propagation
    - REQUIRED : 부모 트랜잭션 없으면 새로운 트랜잭션 생성, 기존 트랜잭션 있으면 에러 시 롤백 전파됨
    - REQUIRES_NEW : 기존 트랜잭션과 완전히 별개의 트랜잭션 생성 - 에러 시 롤백 전파되지 않음
    - NESTED : 부모 트랜잭션의 커밋, 롤백은 영향받고 내 커밋, 롤백은 부모 트랜잭션에 영향을 주지 않음
    - NEVER : 트랜잭션 안씀
    - MANDATORY : 부모 트랜잭션이 없으면 에러