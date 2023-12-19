> Request 진입 시 Spring Security Filter를 타는 순서를 알아보자

### WebAsyncManagerIntegrationFilter
- ThreadLocal을 사용하는 SecurityContext를 비동기 처리 시 생성되는 Thread에서도 사용할 수 있게 처리해준다.
### SecurityContextPersistenceFilter
- SecurityContext를 영속화 하는 필터
    - 요청 전반에 걸쳐 SecurityContext를 사용할 수 있도록 설정하는 filter
### HeaderWriterFilter
- Security 관련 default header 관리를 위한 필터
### CorsFilter
- CORS / SOP ( Same-Origin-Policy ) 관련 처리를 위한 필터 (WebMvcConfigurer, @cors 등)
### LogoutFilter
- Logout request를 받았을 때, 처리해줄 수 있는 filter
### CustomFilter\$\$EnhancerBySpringCGLIB\$\$ceb3ea63
- proxy객체로 만들어지는 Custom Filter
    - SecurityConfig 설정 시, addFilterBefore 등을 통해 특정 필터 동작 이후에 넣을 수 있음
### RequestCacheAwareFilter
- request에 대한 캐시 요청이 있으면 처리해주는 filter
### SecurityContextHolderAwareRequestFilter
- ServletRequest를 servlet api의 보안 메서드를 추가한 wrapper로 감싸는 filter
    - login, logout, authenticate 등의 메서드 사용 가능
### AnonymousAuthenticationFilter
- SecurityContextHolder에 authentication 객체가 있는지 확인 후, 없으면 생성한다.
    - 익명 사용자를 위한 Authentication 설정 (authority - ROLE_ANONYMOUS)
### SessionManagementFilter
- 세션 변조 방지 설정, 세션 생성 전략 설정. 동일 계정에 대한 세션 수 제한 (최대 세션 수 제한 등)
### ExceptionTranslationFilter
- filterChain 내 Authentication, AceessDenied Exception을 처리한다.
- AuthenticationException 발생 시
    - authenticationEntryPoint
    - AbstractSecurityInterceptor
- AccessDeniedException 발생 시
    - 익명 user인지 아닌지 확인하여 익명 user면 authenticationEntryPoint / 익명이 아니면 AccessDeniedHandler
### FilterSecurityInterceptor