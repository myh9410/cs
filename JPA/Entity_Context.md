DB connection pool 설명 - connection pool이 뭔지? connection pool 동작 원리? 이미 connection이 맺어진 경우를 어떻게 알 수 있는지?

- 영속성 컨텍스트와 엔티티 생명주기
    - 비영속 - 영속성 컨텍스트로 등록되지 않은 상태
    - 영속 - 영속성 컨텍스트 관리 받는 상태
    - 준영속 - 영속성 컨텍스트에 저장되었다가 분리된 상태
    - 삭제 - 엔티티가 삭제되어 flush 시 데이터 삭제되는 상태