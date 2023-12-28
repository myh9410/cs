> 카우치베이스 개념 정리

Couchbase와 Memcached

- couchbase는 자체적으로 memcached를 내장 캐시로 사용
- indexing을 지원
- Cluster > Bucket > Collection 의 크기 순서
    - Cluster와 연결 (ip, user-name, user-password로 연결)
        - Cluster 내부의 사용하고자 하는 Bucket을 조회 (bucket의 이름 ex. hiworks-menu 연결)
            - Bucket의 Collection을 조회 (default도 있음)
- In Java, Cluster의 정보를 조회, cluster의 bucket을 조회 하여 bucket의 Collection 조회하는 부분을 config와 bean으로 등록