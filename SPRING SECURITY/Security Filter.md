> Request 진입 시 Spring Security Filter를 타는 순서를 알아보자

---
## Security Filter Chain DEBUG

@EnableWebSecurity의 debug 설정을 true로 바꾸면 (default: false)  
request가 통과하는 Security Filter들을 확인할 수 있다.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.http.HttpStatus;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.HttpStatusEntryPoint;

@Configuration
@EnableWebSecurity(debug = true)
public class SecurityConfig {

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
            .csrf().disable()
            .httpBasic().disable()
            .cors().and()
            .authorizeRequests()
                .antMatchers("/some-role/**").access("hasRole('SOME_ROLE')") //특정 role에 대해서만 허용
                .antMatchers("/allow-all/**").permitAll() //전체 허용
                .anyRequest().denyAll() //그 외에는 전체 비허용
            .and()
            .exceptionHandling().authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED));

    return http.build();
  }
}
```

```text DEBUG결과
#DEBUG 결과
Request received for GET '/users/12':

org.apache.catalina.connector.RequestFacade@741be43e

servletPath:/users/12
pathInfo:null
headers: 
user-agent: PostmanRuntime/7.35.0
accept: */*
postman-token: 
host: localhost:8080
accept-encoding: gzip, deflate, br
connection: keep-alive

Security filter chain: [
  DisableEncodeUrlFilter
  WebAsyncManagerIntegrationFilter
  SecurityContextPersistenceFilter
  HeaderWriterFilter
  CorsFilter
  LogoutFilter
  RequestCacheAwareFilter
  SecurityContextHolderAwareRequestFilter
  AnonymousAuthenticationFilter
  SessionManagementFilter
  ExceptionTranslationFilter
  FilterSecurityInterceptor
]
```
---
## Filter Chain 별 역할

### DisableEncodeUrlFilter
- Session ID가 url에 포함되는것을 방지하기 위해 HttpServletResponse를 사용해 URL encode를 막는 filter
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

---
## CustomFilter를 특정 필터 전, 후에 추가하기

request에 대해 custom filter를 통해 특정 처리들을 추가할 수 있다.   
생성한 custom filter를 어느 단계에서 적용시킬 지 설정할 수 있다.

```java
package com.springboot.mq.common.config.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Objects;

@Slf4j
@Configuration
public class CustomFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            if (Objects.isNull(request.getHeader("some-header"))) {
                log.warn("no header :: some-header");
            } else {
                //do something
            }
        } catch (Exception e) {
            log.error("customFilter Error :: {}", e.getMessage());
        } finally {
            filterChain.doFilter(request, response);
        }
    }
}

public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
          // ...
          .addFilterBefore(customFilter, WebAsyncManagerIntegrationFilter.class)
          // ...
        
        return http.build();
    }
}
```

```text
Security filter chain: [
  DisableEncodeUrlFilter
  CustomFilter$$EnhancerBySpringCGLIB$$516f8f31 <<  원하는 위치에 추가됨
  WebAsyncManagerIntegrationFilter
  SecurityContextPersistenceFilter
  HeaderWriterFilter
  CorsFilter
  LogoutFilter
  RequestCacheAwareFilter
  SecurityContextHolderAwareRequestFilter
  AnonymousAuthenticationFilter
  SessionManagementFilter
  ExceptionTranslationFilter
  FilterSecurityInterceptor
]

```
