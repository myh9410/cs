> 외부 API 장애에 대한 영향을 줄이는 방법에 대해 알아보자

### Time Out
API 연동 시, time out 설정을 통해 대기상태에 머물지 않고, 에러를 뱉을 수 있도록 처리한다.

- Connection TimeOut
  - api 연결에 대한 타임아웃 설정 (usu. 1~5초)
- Read TimeOut
  - 읽기 타임아웃 설정 (usu. 1~3초, 특정 도메인에 따라 시간은 다를 수 있음)

Java의 외부 API 연동을 위한 Libraries의 default 설정 확인해보기
- RestTemplate
  - ClientHttpRequestFactory에 따라 다름
    - ex. apache.http.client.HttpClient 사용 시, Connection과 Read 둘 다 default INF
- HttpClient
  - ?
- WebClient (Spring-Webflux)
  - ?
- FeignClient (Spring-Cloud)
  - ?

### Bulk Head
- BulkHead란?
  - design pattern의 일종 
  - API내에서 Http Connection Pool을 공유하게 되면, 특정 외부 API - A의 장애로 인한 영향이 다른 B, C API 연동 동작에 영향을 미친다.
  - 이러한 외부 API의 장애로 인해 다른 동작에 연쇄적으로 장애가 발생하지 않도록 하기 위해 각 서비스나 기능을 처리하는 Thread Pool을 분리하여 장애가 전파되지 않도록 한다. 

### Circuit Breaker
  - Resilience4j, @Bulkhead
  - https://resilience4j.readme.io/v1.7.0/docs/bulkhead