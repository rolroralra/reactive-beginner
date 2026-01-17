# Filter와 Interceptor

## 1. 개요

Spring WebFlux에서 요청/응답을 가로채는 방법은 여러 가지가 있습니다. 각 방식의 특징과 사용 사례를 알아봅니다.

### 1.1 필터 종류

```
Request → WebFilter → HandlerFilterFunction → Handler → Response
              ↓
         (전역 필터)    (라우터 레벨 필터)
```

| 타입 | 범위 | 사용 위치 |
|------|------|----------|
| WebFilter | 전역 | 모든 요청 |
| HandlerFilterFunction | 라우터 | Functional Endpoints |
| ExchangeFilterFunction | 클라이언트 | WebClient |

## 2. WebFilter (전역 필터)

### 2.1 기본 구현

```java
package com.example.filter;

import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

@Component
public class LoggingWebFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String path = exchange.getRequest().getPath().value();
        String method = exchange.getRequest().getMethod().name();

        log.info("Request: {} {}", method, path);

        return chain.filter(exchange)
            .doOnSuccess(aVoid -> {
                long duration = System.currentTimeMillis() - startTime;
                int status = exchange.getResponse().getStatusCode().value();
                log.info("Response: {} {} - {} ({}ms)", method, path, status, duration);
            })
            .doOnError(error ->
                log.error("Error: {} {} - {}", method, path, error.getMessage()));
    }
}
```

### 2.2 필터 순서 지정

```java
@Component
@Order(1)  // 낮은 숫자가 먼저 실행
public class FirstFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        log.info("First filter - before");
        return chain.filter(exchange)
            .doOnSuccess(v -> log.info("First filter - after"));
    }
}

@Component
@Order(2)
public class SecondFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        log.info("Second filter - before");
        return chain.filter(exchange)
            .doOnSuccess(v -> log.info("Second filter - after"));
    }
}
```

### 2.3 인증 필터

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class AuthenticationFilter implements WebFilter {

    private final JwtService jwtService;

    public AuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 공개 경로는 통과
        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }

        // Authorization 헤더 확인
        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);

        return jwtService.validateToken(token)
            .flatMap(claims -> {
                // 인증 정보를 exchange 속성에 저장
                exchange.getAttributes().put("user", claims.getSubject());
                exchange.getAttributes().put("roles", claims.get("roles"));
                return chain.filter(exchange);
            })
            .onErrorResume(e -> unauthorized(exchange, "Invalid token"));
    }

    private boolean isPublicPath(String path) {
        return path.startsWith("/public") ||
               path.startsWith("/auth/login") ||
               path.startsWith("/auth/register");
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);

        String body = "{\"error\":\"" + message + "\"}";
        DataBuffer buffer = exchange.getResponse().bufferFactory()
            .wrap(body.getBytes(StandardCharsets.UTF_8));

        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

### 2.4 CORS 필터

```java
@Component
public class CorsFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        response.getHeaders().add("Access-Control-Allow-Origin", "*");
        response.getHeaders().add("Access-Control-Allow-Methods",
            "GET, POST, PUT, DELETE, OPTIONS");
        response.getHeaders().add("Access-Control-Allow-Headers",
            "Content-Type, Authorization");
        response.getHeaders().add("Access-Control-Max-Age", "3600");

        // Preflight 요청 처리
        if (request.getMethod() == HttpMethod.OPTIONS) {
            response.setStatusCode(HttpStatus.NO_CONTENT);
            return Mono.empty();
        }

        return chain.filter(exchange);
    }
}
```

### 2.5 요청 ID 추적 필터

```java
@Component
public class RequestIdFilter implements WebFilter {

    private static final String REQUEST_ID_HEADER = "X-Request-ID";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String requestId = exchange.getRequest().getHeaders().getFirst(REQUEST_ID_HEADER);

        if (requestId == null) {
            requestId = UUID.randomUUID().toString();
        }

        // 요청에 ID 저장
        exchange.getAttributes().put("requestId", requestId);

        // 응답 헤더에 ID 추가
        String finalRequestId = requestId;
        exchange.getResponse().beforeCommit(() -> {
            exchange.getResponse().getHeaders().add(REQUEST_ID_HEADER, finalRequestId);
            return Mono.empty();
        });

        // MDC에 설정 (로깅용)
        return chain.filter(exchange)
            .contextWrite(Context.of("requestId", requestId));
    }
}
```

### 2.6 Rate Limiting 필터

```java
@Component
public class RateLimitFilter implements WebFilter {

    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String clientIp = getClientIp(exchange);
        RateLimiter limiter = limiters.computeIfAbsent(clientIp,
            k -> new RateLimiter(100, Duration.ofMinutes(1)));  // 분당 100회

        if (!limiter.tryAcquire()) {
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            exchange.getResponse().getHeaders().add("Retry-After", "60");
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    private String getClientIp(ServerWebExchange exchange) {
        String xff = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
        if (xff != null) {
            return xff.split(",")[0].trim();
        }
        InetSocketAddress remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null ? remoteAddress.getAddress().getHostAddress() : "unknown";
    }
}
```

## 3. HandlerFilterFunction (Functional Endpoints)

### 3.1 기본 사용

```java
@Configuration
public class RouterConfig {

    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler handler) {
        return route()
            .path("/users", builder -> builder
                .GET("/{id}", handler::getUser)
                .GET("", handler::getAllUsers)
                .POST("", handler::createUser)
                .filter(loggingFilter())
                .filter(authFilter())
            )
            .build();
    }

    private HandlerFilterFunction<ServerResponse, ServerResponse> loggingFilter() {
        return (request, next) -> {
            log.info("Request: {} {}", request.method(), request.path());
            long start = System.currentTimeMillis();

            return next.handle(request)
                .doOnSuccess(response -> {
                    long duration = System.currentTimeMillis() - start;
                    log.info("Response: {} ({}ms)", response.statusCode(), duration);
                });
        };
    }

    private HandlerFilterFunction<ServerResponse, ServerResponse> authFilter() {
        return (request, next) -> {
            String auth = request.headers().firstHeader("Authorization");

            if (auth == null || !auth.startsWith("Bearer ")) {
                return ServerResponse.status(HttpStatus.UNAUTHORIZED)
                    .bodyValue(Map.of("error", "Unauthorized"));
            }

            return next.handle(request);
        };
    }
}
```

### 3.2 조건부 필터

```java
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler, AdminHandler adminHandler) {
    HandlerFilterFunction<ServerResponse, ServerResponse> adminFilter =
        (request, next) -> {
            String role = request.headers().firstHeader("X-User-Role");
            if (!"ADMIN".equals(role)) {
                return ServerResponse.status(HttpStatus.FORBIDDEN)
                    .bodyValue(Map.of("error", "Admin access required"));
            }
            return next.handle(request);
        };

    return route()
        // 일반 사용자 API
        .path("/users", builder -> builder
            .GET("", handler::getAllUsers)
            .GET("/{id}", handler::getUser)
        )
        // 관리자 API (필터 적용)
        .path("/admin", builder -> builder
            .GET("/users", adminHandler::getAllUsers)
            .DELETE("/users/{id}", adminHandler::deleteUser)
            .filter(adminFilter)
        )
        .build();
}
```

### 3.3 에러 처리 필터

```java
private HandlerFilterFunction<ServerResponse, ServerResponse> errorHandlingFilter() {
    return (request, next) -> next.handle(request)
        .onErrorResume(NotFoundException.class, e ->
            ServerResponse.notFound().build())
        .onErrorResume(ValidationException.class, e ->
            ServerResponse.badRequest()
                .bodyValue(Map.of("error", e.getMessage(), "errors", e.getErrors())))
        .onErrorResume(Exception.class, e -> {
            log.error("Unexpected error", e);
            return ServerResponse.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .bodyValue(Map.of("error", "Internal server error"));
        });
}
```

## 4. ExchangeFilterFunction (WebClient)

### 4.1 로깅 필터

```java
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .filter(logRequest())
        .filter(logResponse())
        .build();
}

private ExchangeFilterFunction logRequest() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        log.info("Request: {} {}", request.method(), request.url());
        request.headers().forEach((name, values) ->
            values.forEach(value -> log.debug("Header: {}={}", name, value)));
        return Mono.just(request);
    });
}

private ExchangeFilterFunction logResponse() {
    return ExchangeFilterFunction.ofResponseProcessor(response -> {
        log.info("Response: {}", response.statusCode());
        return Mono.just(response);
    });
}
```

### 4.2 인증 토큰 필터

```java
@Bean
public WebClient webClient(TokenService tokenService) {
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .filter(authHeaderFilter(tokenService))
        .build();
}

private ExchangeFilterFunction authHeaderFilter(TokenService tokenService) {
    return (request, next) -> tokenService.getAccessToken()
        .map(token -> ClientRequest.from(request)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .build())
        .flatMap(next::exchange);
}
```

### 4.3 재시도 필터

```java
private ExchangeFilterFunction retryFilter() {
    return (request, next) -> next.exchange(request)
        .flatMap(response -> {
            if (response.statusCode().is5xxServerError()) {
                return Mono.error(new ServerException("Server error"));
            }
            return Mono.just(response);
        })
        .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
            .filter(e -> e instanceof ServerException));
}
```

### 4.4 타임아웃 필터

```java
private ExchangeFilterFunction timeoutFilter(Duration timeout) {
    return (request, next) -> next.exchange(request)
        .timeout(timeout)
        .onErrorMap(TimeoutException.class,
            e -> new ServiceTimeoutException("Request timed out after " + timeout));
}
```

## 5. 필터 체인 구성

```java
@Configuration
public class WebFluxConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .filter(requestIdFilter())     // 1. 요청 ID 추가
            .filter(loggingFilter())       // 2. 요청/응답 로깅
            .filter(authFilter())          // 3. 인증 헤더 추가
            .filter(retryFilter())         // 4. 재시도
            .filter(timeoutFilter())       // 5. 타임아웃
            .build();
    }

    // ... 각 필터 메서드 구현
}
```

## 6. WebFilter vs HandlerFilterFunction

| 특성 | WebFilter | HandlerFilterFunction |
|------|-----------|----------------------|
| 범위 | 전역 | 특정 라우터 |
| 적용 대상 | 모든 요청 | Functional Endpoints |
| 등록 방식 | @Component | RouterFunction.filter() |
| 순서 제어 | @Order | 선언 순서 |
| 사용 사례 | 인증, CORS, 로깅 | 라우터별 검증 |

## 7. 다음 단계

- [Testing](10_Testing.md): 필터 테스트
- [Spring Security 통합](11_Spring_Security_통합.md): 보안 필터
- [예외 처리](08_예외_처리.md): 필터에서의 에러 처리
