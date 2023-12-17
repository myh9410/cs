> HikariCP의 커넥션 요청 동작 방식을 이해해보자


- HikariDataSource.getConnection(); → HikariPool에 커넥션 획득 요청

- HikariPool의 connectionBag.borrow()를 통해 PoolEntry 조회
    - PoolEntry란? Connection 정보, 상태, 접근 시간 등의 정보들을 갖는 객체

- ConnectionBag이 갖는 정보
    
    - connection을 맺을 수 없는 경우, handoffQueue에 PoolEntry를 add, poll하면서 커넥션 관리
        - SynchronousQueue란? put, take 동안 block 처리되는 queue
            - put되면, take 전까지 block 되고 그 반대도 동일한 형식
    
    - ThreadList, SharedList, HandoffQueue
        - ThreadList - ThreadLocal<List<Object>>
            - ThreadLocal의 list 이전에 연결된 정보를 저장하는 용도?
        - SharedList - CopyOnWriteArrayList(thread safe한 array)
            - Connection Pool의 모든 Connection 정보를 갖는 리스트
        - HandoffQueue - SynchronousQueue
            - 커넥션 사용을 위한 대기 큐