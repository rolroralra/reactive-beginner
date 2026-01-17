# Functional Endpoints

## 1. 개요

Functional Endpoints는 Spring WebFlux에서 제공하는 **함수형 프로그래밍 스타일**의 라우팅 방식입니다. 어노테이션 기반과 달리 명시적인 라우팅 정의를 제공합니다.

### 1.1 핵심 구성 요소

```
┌─────────────────────────────────────────────────────────┐
│                  RouterFunction                          │
│           (요청 URL → HandlerFunction 매핑)               │
├─────────────────────────────────────────────────────────┤
│                  HandlerFunction                         │
│         (ServerRequest → Mono<ServerResponse>)          │
├─────────────────────────────────────────────────────────┤
│     ServerRequest              ServerResponse            │
│    (요청 정보 추상화)            (응답 빌더)               │
└─────────────────────────────────────────────────────────┘
```

## 2. 기본 구조

### 2.1 Handler 클래스

```java
package com.example.handler;

import com.example.model.User;
import com.example.service.UserService;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import reactor.core.publisher.Mono;

import static org.springframework.web.reactive.function.server.ServerResponse.*;
import static org.springframework.http.MediaType.*;

@Component
public class UserHandler {

    private final UserService userService;

    public UserHandler(UserService userService) {
        this.userService = userService;
    }

    public Mono<ServerResponse> getUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return userService.findById(id)
            .flatMap(user -> ok().contentType(APPLICATION_JSON).bodyValue(user))
            .switchIfEmpty(notFound().build());
    }

    public Mono<ServerResponse> getAllUsers(ServerRequest request) {
        return ok()
            .contentType(APPLICATION_JSON)
            .body(userService.findAll(), User.class);
    }

    public Mono<ServerResponse> createUser(ServerRequest request) {
        return request.bodyToMono(User.class)
            .flatMap(userService::save)
            .flatMap(user -> created(URI.create("/users/" + user.getId()))
                .contentType(APPLICATION_JSON)
                .bodyValue(user));
    }

    public Mono<ServerResponse> updateUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return request.bodyToMono(User.class)
            .flatMap(user -> {
                user.setId(id);
                return userService.save(user);
            })
            .flatMap(user -> ok().contentType(APPLICATION_JSON).bodyValue(user))
            .switchIfEmpty(notFound().build());
    }

    public Mono<ServerResponse> deleteUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return userService.deleteById(id)
            .then(noContent().build());
    }
}
```

### 2.2 Router 설정

```java
package com.example.router;

import com.example.handler.UserHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.ServerResponse;

import static org.springframework.web.reactive.function.server.RouterFunctions.route;
import static org.springframework.web.reactive.function.server.RequestPredicates.*;
import static org.springframework.http.MediaType.*;

@Configuration
public class RouterConfig {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return route()
            .path("/users", builder -> builder
                .GET("/{id}", handler::getUser)
                .GET("", handler::getAllUsers)
                .POST("", accept(APPLICATION_JSON), handler::createUser)
                .PUT("/{id}", accept(APPLICATION_JSON), handler::updateUser)
                .DELETE("/{id}", handler::deleteUser)
            )
            .build();
    }
}
```

## 3. RouterFunction 상세

### 3.1 라우팅 정의 방법

```java
// 방법 1: route() 빌더 사용
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler) {
    return route()
        .GET("/users", handler::getAllUsers)
        .GET("/users/{id}", handler::getUser)
        .build();
}

// 방법 2: RouterFunctions.route() 사용
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler) {
    return RouterFunctions
        .route(GET("/users"), handler::getAllUsers)
        .andRoute(GET("/users/{id}"), handler::getUser);
}

// 방법 3: nest()로 그룹핑
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler) {
    return route()
        .nest(path("/api/v1"), builder -> builder
            .nest(path("/users"), b -> b
                .GET("", handler::getAllUsers)
                .GET("/{id}", handler::getUser)
                .POST("", handler::createUser)
            )
            .nest(path("/orders"), b -> b
                .GET("", orderHandler::getAllOrders)
            )
        )
        .build();
}
```

### 3.2 RequestPredicate 조합

```java
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler) {
    return route()
        // Content-Type 조건
        .POST("/users", accept(APPLICATION_JSON), handler::createUser)

        // Accept 헤더 조건
        .GET("/users", accept(APPLICATION_JSON), handler::getAllUsersJson)
        .GET("/users", accept(APPLICATION_XML), handler::getAllUsersXml)

        // 여러 조건 조합
        .PUT("/users/{id}",
            accept(APPLICATION_JSON).and(contentType(APPLICATION_JSON)),
            handler::updateUser)

        // 헤더 조건
        .GET("/admin/users",
            headers(h -> h.header("X-Admin-Key").equals("secret")),
            handler::getAdminUsers)

        // 쿼리 파라미터 조건
        .GET("/users",
            queryParam("status", "active"::equals),
            handler::getActiveUsers)

        .build();
}
```

### 3.3 여러 Router 조합

```java
@Configuration
public class RouterConfig {

    @Bean
    public RouterFunction<ServerResponse> combinedRoutes(
            UserHandler userHandler,
            OrderHandler orderHandler,
            ProductHandler productHandler) {

        return route()
            .add(userRoutes(userHandler))
            .add(orderRoutes(orderHandler))
            .add(productRoutes(productHandler))
            .build();
    }

    private RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return route()
            .path("/users", builder -> builder
                .GET("/{id}", handler::getUser)
                .GET("", handler::getAllUsers)
            )
            .build();
    }

    private RouterFunction<ServerResponse> orderRoutes(OrderHandler handler) {
        return route()
            .path("/orders", builder -> builder
                .GET("/{id}", handler::getOrder)
                .GET("", handler::getAllOrders)
            )
            .build();
    }

    private RouterFunction<ServerResponse> productRoutes(ProductHandler handler) {
        return route()
            .path("/products", builder -> builder
                .GET("/{id}", handler::getProduct)
                .GET("", handler::getAllProducts)
            )
            .build();
    }
}
```

## 4. ServerRequest

### 4.1 Path Variable

```java
public Mono<ServerResponse> getUser(ServerRequest request) {
    // 문자열로 가져오기
    String id = request.pathVariable("id");

    // Long으로 변환
    Long userId = Long.parseLong(request.pathVariable("id"));

    return userService.findById(userId)
        .flatMap(user -> ok().bodyValue(user));
}
```

### 4.2 Query Parameter

```java
public Mono<ServerResponse> searchUsers(ServerRequest request) {
    // 단일 파라미터
    Optional<String> name = request.queryParam("name");

    // 기본값
    String status = request.queryParam("status").orElse("active");

    // 다중 값
    List<String> tags = request.queryParams().get("tag");

    // 모든 파라미터
    MultiValueMap<String, String> params = request.queryParams();

    return ok().body(userService.search(name, status, tags), User.class);
}
```

### 4.3 Request Body

```java
// Mono로 단일 객체
public Mono<ServerResponse> createUser(ServerRequest request) {
    return request.bodyToMono(User.class)
        .flatMap(userService::save)
        .flatMap(user -> created(URI.create("/users/" + user.getId())).bodyValue(user));
}

// Flux로 스트림
public Mono<ServerResponse> createUsers(ServerRequest request) {
    return request.bodyToFlux(User.class)
        .flatMap(userService::save)
        .collectList()
        .flatMap(users -> ok().bodyValue(users));
}

// Form 데이터
public Mono<ServerResponse> submitForm(ServerRequest request) {
    return request.formData()
        .flatMap(formData -> {
            String username = formData.getFirst("username");
            String password = formData.getFirst("password");
            return authService.authenticate(username, password);
        })
        .flatMap(user -> ok().bodyValue(user));
}

// Multipart 데이터
public Mono<ServerResponse> uploadFile(ServerRequest request) {
    return request.multipartData()
        .flatMap(parts -> {
            Part filePart = parts.getFirst("file");
            if (filePart instanceof FilePart) {
                return fileService.save((FilePart) filePart);
            }
            return Mono.error(new IllegalArgumentException("No file"));
        })
        .flatMap(filename -> ok().bodyValue("Uploaded: " + filename));
}
```

### 4.4 Headers & Cookies

```java
public Mono<ServerResponse> getInfo(ServerRequest request) {
    // 헤더
    ServerRequest.Headers headers = request.headers();
    List<String> auth = headers.header("Authorization");
    Optional<MediaType> contentType = headers.contentType();
    List<MediaType> accept = headers.accept();

    // 단일 헤더
    String customHeader = headers.firstHeader("X-Custom-Header");

    // 쿠키
    MultiValueMap<String, HttpCookie> cookies = request.cookies();
    HttpCookie sessionCookie = cookies.getFirst("sessionId");

    return ok().bodyValue("OK");
}
```

### 4.5 기타 정보

```java
public Mono<ServerResponse> getRequestInfo(ServerRequest request) {
    // HTTP 메서드
    HttpMethod method = request.method();

    // URI 정보
    URI uri = request.uri();
    String path = request.path();

    // 원격 주소
    Optional<InetSocketAddress> remoteAddress = request.remoteAddress();

    // Principal (인증 정보)
    Mono<Principal> principal = request.principal();

    // Exchange (저수준 접근)
    ServerWebExchange exchange = request.exchange();

    return ok().bodyValue(Map.of(
        "method", method.name(),
        "path", path,
        "uri", uri.toString()
    ));
}
```

## 5. ServerResponse

### 5.1 상태 코드

```java
// 200 OK
ServerResponse.ok()

// 201 Created
ServerResponse.created(URI.create("/users/1"))

// 202 Accepted
ServerResponse.accepted()

// 204 No Content
ServerResponse.noContent()

// 301 Permanent Redirect
ServerResponse.permanentRedirect(URI.create("/new-location"))

// 302 Temporary Redirect
ServerResponse.temporaryRedirect(URI.create("/temp-location"))

// 304 Not Modified
ServerResponse.status(HttpStatus.NOT_MODIFIED)

// 400 Bad Request
ServerResponse.badRequest()

// 404 Not Found
ServerResponse.notFound()

// 500 Internal Server Error
ServerResponse.status(HttpStatus.INTERNAL_SERVER_ERROR)

// 커스텀 상태 코드
ServerResponse.status(418)  // I'm a teapot
```

### 5.2 Body 설정

```java
// 단일 값
return ok().bodyValue(user);

// Mono
return ok().body(userMono, User.class);

// Flux
return ok().body(userFlux, User.class);

// 빈 응답
return noContent().build();

// BodyInserters 사용
return ok().body(BodyInserters.fromValue(user));
return ok().body(BodyInserters.fromPublisher(userMono, User.class));
```

### 5.3 Headers & Cookies

```java
public Mono<ServerResponse> response(ServerRequest request) {
    return ok()
        // Content-Type
        .contentType(MediaType.APPLICATION_JSON)

        // 커스텀 헤더
        .header("X-Custom-Header", "value")
        .headers(headers -> {
            headers.add("X-Header-1", "value1");
            headers.add("X-Header-2", "value2");
        })

        // 쿠키
        .cookie(ResponseCookie.from("sessionId", "abc123")
            .httpOnly(true)
            .secure(true)
            .path("/")
            .maxAge(Duration.ofHours(1))
            .build())

        // Cache-Control
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(30)))

        // ETag
        .eTag("\"v1\"")

        // Last-Modified
        .lastModified(ZonedDateTime.now())

        .bodyValue(data);
}
```

### 5.4 스트리밍 응답

```java
// Server-Sent Events
public Mono<ServerResponse> streamEvents(ServerRequest request) {
    Flux<ServerSentEvent<String>> events = Flux.interval(Duration.ofSeconds(1))
        .map(seq -> ServerSentEvent.<String>builder()
            .id(String.valueOf(seq))
            .event("message")
            .data("Event " + seq)
            .build());

    return ok()
        .contentType(MediaType.TEXT_EVENT_STREAM)
        .body(events, ServerSentEvent.class);
}

// NDJSON 스트리밍
public Mono<ServerResponse> streamUsers(ServerRequest request) {
    return ok()
        .contentType(MediaType.APPLICATION_NDJSON)
        .body(userService.findAll(), User.class);
}
```

### 5.5 파일 응답

```java
public Mono<ServerResponse> downloadFile(ServerRequest request) {
    String filename = request.pathVariable("filename");
    Resource resource = new FileSystemResource(Path.of("uploads", filename));

    return ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + filename + "\"")
        .body(BodyInserters.fromResource(resource));
}
```

## 6. 필터 적용

### 6.1 Router 레벨 필터

```java
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler) {
    return route()
        .path("/users", builder -> builder
            .GET("/{id}", handler::getUser)
            .GET("", handler::getAllUsers)
            .filter(loggingFilter())
        )
        .build();
}

private HandlerFilterFunction<ServerResponse, ServerResponse> loggingFilter() {
    return (request, next) -> {
        log.info("Request: {} {}", request.method(), request.path());
        long startTime = System.currentTimeMillis();

        return next.handle(request)
            .doOnSuccess(response -> {
                long duration = System.currentTimeMillis() - startTime;
                log.info("Response: {} ({}ms)", response.statusCode(), duration);
            });
    };
}
```

### 6.2 인증 필터

```java
private HandlerFilterFunction<ServerResponse, ServerResponse> authFilter() {
    return (request, next) -> {
        String authHeader = request.headers().firstHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return ServerResponse.status(HttpStatus.UNAUTHORIZED)
                .bodyValue("Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);
        return authService.validateToken(token)
            .flatMap(valid -> {
                if (valid) {
                    return next.handle(request);
                }
                return ServerResponse.status(HttpStatus.FORBIDDEN)
                    .bodyValue("Invalid token");
            });
    };
}

@Bean
public RouterFunction<ServerResponse> securedRoutes(UserHandler handler) {
    return route()
        .path("/api/secure", builder -> builder
            .GET("/users", handler::getAllUsers)
            .filter(authFilter())
        )
        .build();
}
```

### 6.3 예외 처리 필터

```java
private HandlerFilterFunction<ServerResponse, ServerResponse> errorHandler() {
    return (request, next) -> next.handle(request)
        .onErrorResume(IllegalArgumentException.class, e ->
            ServerResponse.badRequest().bodyValue(Map.of("error", e.getMessage())))
        .onErrorResume(NotFoundException.class, e ->
            ServerResponse.notFound().build())
        .onErrorResume(Exception.class, e ->
            ServerResponse.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .bodyValue(Map.of("error", "Internal Server Error")));
}
```

## 7. 유효성 검사

```java
@Component
public class UserHandler {

    private final Validator validator;
    private final UserService userService;

    public UserHandler(Validator validator, UserService userService) {
        this.validator = validator;
        this.userService = userService;
    }

    public Mono<ServerResponse> createUser(ServerRequest request) {
        return request.bodyToMono(CreateUserRequest.class)
            .flatMap(this::validate)
            .flatMap(req -> userService.save(req.toUser()))
            .flatMap(user -> created(URI.create("/users/" + user.getId()))
                .bodyValue(user))
            .onErrorResume(ValidationException.class, e ->
                badRequest().bodyValue(Map.of("errors", e.getErrors())));
    }

    private Mono<CreateUserRequest> validate(CreateUserRequest request) {
        Set<ConstraintViolation<CreateUserRequest>> violations =
            validator.validate(request);

        if (!violations.isEmpty()) {
            List<String> errors = violations.stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .collect(Collectors.toList());
            return Mono.error(new ValidationException(errors));
        }

        return Mono.just(request);
    }
}
```

## 8. 테스트

```java
@WebFluxTest
@Import({RouterConfig.class, UserHandler.class})
class UserRoutesTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private UserService userService;

    @Test
    void shouldGetUser() {
        User user = new User(1L, "홍길동", "hong@example.com");
        when(userService.findById(1L)).thenReturn(Mono.just(user));

        webTestClient.get()
            .uri("/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .isEqualTo(user);
    }

    @Test
    void shouldReturnNotFoundForMissingUser() {
        when(userService.findById(999L)).thenReturn(Mono.empty());

        webTestClient.get()
            .uri("/users/999")
            .exchange()
            .expectStatus().isNotFound();
    }
}
```

## 9. 다음 단계

- [WebClient](05_WebClient.md): 리액티브 HTTP 클라이언트
- [예외 처리](08_예외_처리.md): 에러 핸들링
- [Filter와 Interceptor](09_Filter와_Interceptor.md): 필터 상세 가이드
