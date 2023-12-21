> Web Request가 Spring Boot에서 처리되는 방식에 대해 알아보자

- Servlet이란?
    - client의 request를 처리하고, response를 반환하는 JAVA의 웹 프로그래밍 기술
- HttpRequest 객체 > 서블릿 컨테이너
- DispatcherServlet
    1. client의 요청을 DispatcherServlet에서 받음
    2. 요청을 위임할 controller를 조회
    3. 요청을 위임하는 역할을 하는 Handler Adapter를 찾아서 전달
    4. Handler Adapter가 controller로 요청 위임
        1. MappingJackson2HttpMessageConverter 등을 통해 프레임워크 내에서 사용하도록 매핑
    5. 비즈니스 로직 처리
    6. controller에서 response 반환
    7. Handler Adapter가 response를 DispatcherServlet으로 전달
    8. DispatcherServlet이 response를 client에게 전달
- securityFilterChain
    - request에 적용할 security filter들에 대해 정의하는 configuration 설정을 통해 구축되는 chain