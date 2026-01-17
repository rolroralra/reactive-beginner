# Spring WebFlux 개요

## 1. Spring WebFlux란?

Spring WebFlux는 Spring Framework 5에서 도입된 **리액티브 웹 프레임워크**입니다. 기존의 Spring MVC와 달리 **Non-Blocking I/O**와 **Reactive Streams** 기반으로 동작하여, 적은 리소스로 높은 동시성을 처리할 수 있습니다.

```
┌─────────────────────────────────────────────────────────────┐
│                     Spring Framework 5                       │
├─────────────────────────────┬───────────────────────────────┤
│        Spring MVC           │        Spring WebFlux          │
│    (Servlet 기반)            │     (Reactive 기반)            │
│    Blocking I/O             │     Non-Blocking I/O           │
│    Thread per Request       │     Event Loop                 │
├─────────────────────────────┼───────────────────────────────┤
│         Tomcat              │    Netty / Undertow            │
│         Jetty               │    Tomcat / Jetty              │
└─────────────────────────────┴───────────────────────────────┘
```

## 2. Spring MVC vs Spring WebFlux

### 2.1 아키텍처 비교

| 특성 | Spring MVC | Spring WebFlux |
|------|-----------|----------------|
| **I/O 모델** | Blocking | Non-Blocking |
| **프로그래밍 모델** | 명령형 (Imperative) | 반응형 (Reactive) |
| **기본 서버** | Tomcat (Servlet) | Netty (Non-Servlet) |
| **동시성 처리** | Thread per Request | Event Loop |
| **리턴 타입** | Object, ResponseEntity | Mono, Flux |
| **JDBC 지원** | O (동기) | X (R2DBC 사용) |

### 2.2 Thread 모델 비교

**Spring MVC (Thread per Request)**
```
Request 1 ──→ Thread 1 ──→ DB 조회 (Blocking) ──→ Response
Request 2 ──→ Thread 2 ──→ DB 조회 (Blocking) ──→ Response
Request 3 ──→ Thread 3 ──→ DB 조회 (Blocking) ──→ Response
         ...
Request N ──→ Thread N ──→ 대기 (Thread Pool 고갈)
```

**Spring WebFlux (Event Loop)**
```
Request 1 ─┐
Request 2 ─┼──→ Event Loop ──→ Non-Blocking 처리 ──→ Response
Request 3 ─┤    (소수의 Thread)
    ...    │
Request N ─┘
```

## 3. 언제 WebFlux를 사용해야 할까?

### 3.1 WebFlux가 적합한 경우

- **높은 동시성**이 필요한 서비스 (채팅, 알림, 스트리밍)
- **마이크로서비스 아키텍처**에서 다수의 외부 서비스 호출
- **실시간 데이터 처리** (Server-Sent Events, WebSocket)
- **I/O 바운드 작업**이 많은 애플리케이션
- **리소스 효율성**이 중요한 클라우드 환경

### 3.2 Spring MVC가 더 나은 경우

- **CPU 집약적인 작업**이 많은 경우
- **기존 Blocking 라이브러리**에 의존하는 경우 (JDBC, JPA)
- **팀이 리액티브 프로그래밍**에 익숙하지 않은 경우
- **단순한 CRUD** 애플리케이션

### 3.3 혼합 사용

Spring Boot에서는 MVC와 WebFlux를 혼합하여 사용할 수 없습니다. 단, **WebClient**는 MVC 애플리케이션에서도 사용 가능합니다.

```java
// Spring MVC에서 WebClient 사용
@RestController
public class MvcController {

    private final WebClient webClient = WebClient.create();

    @GetMapping("/hybrid")
    public Mono<String> hybridCall() {
        return webClient.get()
            .uri("https://api.example.com/data")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

## 4. Reactive Streams와의 관계

Spring WebFlux는 **Reactive Streams** 표준을 기반으로 합니다.

```
┌─────────────────────────────────────────────────────────┐
│                   Reactive Streams                       │
│         (Publisher, Subscriber, Subscription)            │
├─────────────────────────────────────────────────────────┤
│                    Project Reactor                       │
│                    (Mono, Flux)                          │
├─────────────────────────────────────────────────────────┤
│                   Spring WebFlux                         │
│    (WebHandler, RouterFunction, WebClient)               │
└─────────────────────────────────────────────────────────┘
```

### 4.1 핵심 타입

```java
// Mono: 0 또는 1개의 요소를 비동기적으로 방출
Mono<User> findById(Long id);

// Flux: 0 ~ N개의 요소를 비동기적으로 방출
Flux<User> findAll();
```

## 5. 지원 서버

Spring WebFlux는 다양한 서버에서 실행 가능합니다.

| 서버 | Servlet 기반 | 기본값 |
|------|-------------|--------|
| **Netty** | X | O (기본) |
| **Undertow** | X | X |
| **Tomcat** | O | X |
| **Jetty** | O | X |

### 5.1 서버 변경 예시

```xml
<!-- Netty 대신 Tomcat 사용 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-reactor-netty</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
</dependency>
```

## 6. 프로그래밍 모델

Spring WebFlux는 두 가지 프로그래밍 모델을 지원합니다.

### 6.1 Annotated Controllers

기존 Spring MVC와 유사한 어노테이션 기반 방식입니다.

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public Flux<User> getAllUsers() {
        return userService.findAll();
    }
}
```

### 6.2 Functional Endpoints

함수형 프로그래밍 스타일의 라우팅 방식입니다.

```java
@Configuration
public class RouterConfig {

    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/users/{id}", handler::getUser)
            .GET("/users", handler::getAllUsers)
            .POST("/users", handler::createUser)
            .build();
    }
}
```

## 7. 핵심 컴포넌트

```
┌─────────────────────────────────────────────────────────┐
│                    HttpHandler                           │
│            (서버별 추상화 레이어)                          │
├─────────────────────────────────────────────────────────┤
│                    WebHandler                            │
│         (필터, 예외 처리, 코덱 등)                         │
├─────────────────────────────────────────────────────────┤
│     DispatcherHandler          RouterFunction           │
│   (Annotated Controllers)    (Functional Endpoints)     │
└─────────────────────────────────────────────────────────┘
```

### 7.1 주요 컴포넌트

- **HttpHandler**: 서버별 HTTP 처리 추상화
- **WebHandler**: 필터 체인, 세션, 코덱 등 웹 처리
- **DispatcherHandler**: 어노테이션 기반 컨트롤러 처리
- **RouterFunction**: 함수형 엔드포인트 라우팅

## 8. 다음 단계

- [시작하기](02_시작하기.md): 프로젝트 설정 및 첫 번째 애플리케이션
- [Annotated Controllers](03_Annotated_Controllers.md): 어노테이션 기반 컨트롤러
- [Functional Endpoints](04_Functional_Endpoints.md): 함수형 엔드포인트
