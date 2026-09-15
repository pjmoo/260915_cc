---
name: spring-boot-4
description: 이 프로젝트(Spring Boot 4.1.x, Spring Security 7, Jackson 3, Spring AI 2.x)의 코드를 작성/수정/리뷰할 때 항상 참고한다. Spring Boot 3 시절 문법(antMatchers, authorizeRequests, javax.*, spring-boot-starter-web 등)을 쓰지 않도록 하고, Boot 4의 최신 문법과 주요 breaking change를 정리한다.
---

# Spring Boot 4 최신 문법 & 주요 변경사항

이 프로젝트는 `build.gradle` 기준 **Spring Boot 4.1.1**, Java 17 toolchain, Spring AI 2.0.1을 사용한다.
Spring Boot 3.x 시절 습관으로 코드를 작성하면 컴파일이 안 되거나 deprecated API를 쓰게 되므로,
아래 규칙을 코드 작성/리뷰 시 항상 적용한다.

## 1. 의존성/스타터 이름 변경

- `spring-boot-starter-web` → **`spring-boot-starter-webmvc`** (Spring MVC 전용, WebFlux와 이름으로 구분).
- HTTP 클라이언트가 필요하면 `spring-boot-starter-restclient`(RestClient) 또는 `spring-boot-starter-webclient`(WebClient)를 별도로 추가한다. RestTemplate 자동설정은 더 이상 web 스타터에 묶여 있지 않다.
- 각 스타터에 대응하는 `-test` 스타터가 분리되어 있다 (`spring-boot-starter-webmvc-test`, `spring-boot-starter-security-test`, `spring-boot-starter-data-jpa-test` 등). 테스트 의존성을 추가할 때는 이 패턴을 따른다.
- Boot 코드베이스 자체가 모듈화되어 `org.springframework.boot.<module>` 패키지로 세분화됐다. Boot 내부 패키지에 직접 의존하는 코드는 피한다.

## 2. Jakarta / Jackson / Hibernate

- `javax.*` 패키지는 전부 `jakarta.*`로 대체됐다 (Jakarta EE 11 기반). `javax.persistence`, `javax.validation` 등을 쓰지 않는다.
- Jackson이 **Jackson 3**로 올라가면서 breaking change가 있다. `com.fasterxml.jackson.databind.ObjectMapper` API 일부가 바뀌었으므로, 커스텀 직렬화 코드 작성 시 Jackson 3 문서를 기준으로 확인한다.
- JPA는 **Hibernate 7** 기반이다. `application.yaml`의 `spring.jpa.hibernate.ddl-auto: validate` 같은 설정은 그대로 유효하지만, 엔티티/매핑 작성 시 Hibernate 7 변경사항(예: 일부 deprecated 매핑 애노테이션)을 확인한다.

## 3. Null Safety: JSpecify

- Spring 자체 `@Nullable`/`@NonNull` 애노테이션은 deprecated. **JSpecify** (`org.jspecify.annotations.Nullable` 등)로 대체됐다.
- 새 패키지를 만들 때 `package-info.java`에 `@NullMarked`를 선언하고, nullable한 파라미터/리턴값에만 JSpecify `@Nullable`을 붙이는 패턴을 권장한다.

```java
// package-info.java
@NullMarked
package org.example.cc.web;

import org.jspecify.annotations.NullMarked;
```

## 4. Spring Security 7 (이 프로젝트의 `SecurityConfig` 관련, 중요)

- `authorizeRequests()` → **`authorizeHttpRequests()`**, `antMatchers()` → **`requestMatchers()`**. 구 API는 Security 7에서 완전히 제거됨. (현재 `SecurityConfig.java`는 이미 이 방식을 따르고 있음 — 유지할 것.)
- Security 7 철학은 "암묵적 동작 없음(no implicit behavior)". CSRF, 세션 정책 등은 명시적으로 선언해야 한다.
  - stateless REST API(JWT/OAuth2 bearer 토큰만 사용, 세션 쿠키 없음)라면 CSRF를 명시적으로 비활성화해야 한다: `.csrf(csrf -> csrf.disable())`.
  - 세션을 쓰는 폼/OAuth2 로그인(현재 프로젝트처럼 `oauth2Login` + `logoutSuccessUrl`을 쓰는 경우)은 CSRF를 끄지 않는 것이 기본이며, 끌 필요가 없다.
- OAuth2 클라이언트 PKCE 기본값이 `requireProofKey=false → true`로 바뀌었다. Google OAuth2 로그인 연동 시 클라이언트가 PKCE를 지원하지 않으면 인증 실패가 발생할 수 있으므로, 문제가 생기면 이 기본값 변경을 먼저 의심한다.
- `NimbusJwtDecoder`의 `typ` 헤더 검증이 `JwtTypeValidator`로 이동했다. JWT 리소스 서버를 구성할 경우 관련 커스터마이징 위치가 달라졌음을 인지한다.

## 5. 선언적 HTTP 클라이언트 (`@HttpExchange`)

외부 API 호출 코드를 작성할 때 `RestTemplate`을 새로 만들지 말고, HTTP Interface 방식을 우선 사용한다.

```java
@HttpExchange("/todos")
public interface TodoClient {
    @GetExchange
    List<Todo> findAll();

    @GetExchange("/{id}")
    Todo findById(@PathVariable Long id);

    @PostExchange
    Todo create(@RequestBody Todo todo);
}
```

```java
@Bean
public TodoClient todoClient(RestClient.Builder builder) {
    RestClient client = builder.baseUrl("https://example.com").build();
    HttpServiceProxyFactory factory = HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(client))
        .build();
    return factory.createClient(TodoClient.class);
}
```

## 6. 신규 기능 (필요할 때 적극 활용)

- **API 버저닝**: `@RequestMapping`에 버전 정보를 부여하는 1급 API 버저닝 지원 (path/header/query-param/media-type 전략). REST API 버전 관리가 필요해지면 별도 라이브러리 없이 이 기능을 우선 검토한다.
- **Resilience 애노테이션**: `@Retryable`(재시도 횟수, delay, jitter, backoff 설정), `@ConcurrencyLimit`(동시 호출 수 제한). 외부 API 연동(예: Google GenAI 호출) 코드에 재시도 로직이 필요하면 별도 AOP/라이브러리 대신 이 애노테이션을 우선 검토한다.
- **REST Test Client**: MVC 테스트 작성 시 `RestTestClient`를 활용해 보일러플레이트를 줄인다.
- **Spring Data AOT**: 리포지토리 쿼리를 컴파일 타임에 생성해 시작 시간을 단축한다. 별도 설정 없이 자동 적용되는 영역이므로, 런타임 쿼리 생성 관련 workaround 코드를 새로 만들 필요는 없다.

## 7. 이 프로젝트에 적용 시 체크리스트

1. 새 REST API 컨트롤러/설정을 작성할 때 `spring-boot-starter-webmvc` 기준 클래스(`@RestController`, `HttpServletRequest` 등)를 사용하고, `web` 스타터 시절 클래스명을 혼동하지 않는다.
2. Security 관련 코드를 건드릴 때는 `authorizeHttpRequests`/`requestMatchers`만 사용하고, CSRF/세션 정책을 항상 명시적으로 검토한다.
3. 외부 API 연동 코드는 `RestTemplate` 대신 `RestClient` + `@HttpExchange` 인터페이스로 작성한다.
4. 재시도가 필요한 외부 호출(예: Spring AI 모델 호출)에는 `@Retryable`을 우선 적용한다.
5. 새 패키지 생성 시 `package-info.java` + `@NullMarked` + JSpecify `@Nullable`을 적용한다.
6. `javax.*` import가 보이면 즉시 `jakarta.*`로 교정한다.
