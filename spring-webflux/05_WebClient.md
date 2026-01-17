# WebClient

## 1. 개요

WebClient는 Spring WebFlux에서 제공하는 **Non-Blocking, Reactive HTTP 클라이언트**입니다. RestTemplate의 리액티브 대안으로, 비동기 방식으로 HTTP 요청을 처리합니다.

### 1.1 RestTemplate vs WebClient

| 특성 | RestTemplate | WebClient |
|------|-------------|-----------|
| **I/O 모델** | Blocking | Non-Blocking |
| **스레드** | 요청당 스레드 | 이벤트 루프 |
| **지원 상태** | 유지보수 모드 | 활발한 개발 |
| **스트리밍** | 제한적 | 완전 지원 |
| **Spring MVC 사용** | O | O |
| **Spring WebFlux 사용** | X | O |

## 2. 의존성 및 생성

### 2.1 의존성

WebFlux 스타터에 포함되어 있습니다.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

### 2.2 WebClient 생성

```java
// 기본 생성
WebClient client = WebClient.create();

// Base URL 지정
WebClient client = WebClient.create("https://api.example.com");

// Builder 사용
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
    .build();
```

### 2.3 Bean으로 등록

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(logRequest())
            .filter(logResponse())
            .build();
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.info("Request: {} {}", request.method(), request.url());
            return Mono.just(request);
        });
    }

    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            log.info("Response: {}", response.statusCode());
            return Mono.just(response);
        });
    }
}
```

## 3. GET 요청

### 3.1 기본 GET 요청

```java
// 단일 객체 조회
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);

// 리스트 조회
Flux<User> users = webClient.get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(User.class);

// 결과 사용
user.subscribe(u -> log.info("User: {}", u));
```

### 3.2 Query Parameter

```java
// URI 빌더 사용
Flux<User> users = webClient.get()
    .uri(uriBuilder -> uriBuilder
        .path("/users")
        .queryParam("page", 0)
        .queryParam("size", 10)
        .queryParam("sort", "name,asc")
        .build())
    .retrieve()
    .bodyToFlux(User.class);

// UriComponentsBuilder 사용
URI uri = UriComponentsBuilder.fromPath("/users")
    .queryParam("status", "active")
    .queryParam("role", "admin", "user")  // 다중 값
    .build()
    .toUri();

Flux<User> users = webClient.get()
    .uri(uri)
    .retrieve()
    .bodyToFlux(User.class);
```

### 3.3 헤더 설정

```java
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
    .header("X-Custom-Header", "value")
    .headers(headers -> {
        headers.add("X-Header-1", "value1");
        headers.add("X-Header-2", "value2");
    })
    .accept(MediaType.APPLICATION_JSON)
    .acceptCharset(StandardCharsets.UTF_8)
    .retrieve()
    .bodyToMono(User.class);
```

## 4. POST 요청

### 4.1 JSON Body

```java
User newUser = new User("홍길동", "hong@example.com");

Mono<User> createdUser = webClient.post()
    .uri("/users")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(newUser)
    .retrieve()
    .bodyToMono(User.class);

// Mono로 Body 전달
Mono<User> userMono = Mono.just(newUser);

Mono<User> createdUser = webClient.post()
    .uri("/users")
    .contentType(MediaType.APPLICATION_JSON)
    .body(userMono, User.class)
    .retrieve()
    .bodyToMono(User.class);
```

### 4.2 Form Data

```java
Mono<String> response = webClient.post()
    .uri("/login")
    .contentType(MediaType.APPLICATION_FORM_URLENCODED)
    .body(BodyInserters.fromFormData("username", "user")
        .with("password", "pass"))
    .retrieve()
    .bodyToMono(String.class);

// MultiValueMap 사용
MultiValueMap<String, String> formData = new LinkedMultiValueMap<>();
formData.add("username", "user");
formData.add("password", "pass");

Mono<String> response = webClient.post()
    .uri("/login")
    .contentType(MediaType.APPLICATION_FORM_URLENCODED)
    .bodyValue(formData)
    .retrieve()
    .bodyToMono(String.class);
```

### 4.3 Multipart (파일 업로드)

```java
// Resource로 파일 업로드
Resource fileResource = new FileSystemResource("path/to/file.pdf");

Mono<String> response = webClient.post()
    .uri("/upload")
    .contentType(MediaType.MULTIPART_FORM_DATA)
    .body(BodyInserters.fromMultipartData("file", fileResource)
        .with("description", "My File"))
    .retrieve()
    .bodyToMono(String.class);

// MultipartBodyBuilder 사용
MultipartBodyBuilder builder = new MultipartBodyBuilder();
builder.part("file", fileResource);
builder.part("name", "document.pdf");
builder.part("metadata", Map.of("author", "홍길동"));

Mono<String> response = webClient.post()
    .uri("/upload")
    .contentType(MediaType.MULTIPART_FORM_DATA)
    .body(BodyInserters.fromMultipartData(builder.build()))
    .retrieve()
    .bodyToMono(String.class);
```

## 5. PUT, PATCH, DELETE 요청

### 5.1 PUT 요청

```java
User updatedUser = new User(1L, "홍길동", "hong.updated@example.com");

Mono<User> result = webClient.put()
    .uri("/users/{id}", 1)
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(updatedUser)
    .retrieve()
    .bodyToMono(User.class);
```

### 5.2 PATCH 요청

```java
Map<String, Object> patch = Map.of("email", "new@example.com");

Mono<User> result = webClient.patch()
    .uri("/users/{id}", 1)
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(patch)
    .retrieve()
    .bodyToMono(User.class);
```

### 5.3 DELETE 요청

```java
Mono<Void> result = webClient.delete()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(Void.class);

// 응답 Body가 있는 경우
Mono<DeleteResponse> result = webClient.delete()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(DeleteResponse.class);
```

## 6. 응답 처리

### 6.1 retrieve() vs exchange()

```java
// retrieve(): 간단한 응답 처리 (권장)
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);

// exchangeToMono(): 상세한 응답 제어
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .exchangeToMono(response -> {
        if (response.statusCode().is2xxSuccessful()) {
            return response.bodyToMono(User.class);
        } else if (response.statusCode().is4xxClientError()) {
            return Mono.error(new ClientException("Client error"));
        } else {
            return Mono.error(new ServerException("Server error"));
        }
    });
```

### 6.2 ResponseEntity 사용

```java
Mono<ResponseEntity<User>> responseEntity = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .toEntity(User.class);

responseEntity.subscribe(entity -> {
    HttpStatusCode status = entity.getStatusCode();
    HttpHeaders headers = entity.getHeaders();
    User body = entity.getBody();

    log.info("Status: {}", status);
    log.info("Headers: {}", headers);
    log.info("Body: {}", body);
});

// Flux로 리스트
Mono<ResponseEntity<List<User>>> responseEntity = webClient.get()
    .uri("/users")
    .retrieve()
    .toEntityList(User.class);
```

### 6.3 스트리밍 응답

```java
// Server-Sent Events 수신
Flux<ServerSentEvent<String>> events = webClient.get()
    .uri("/events")
    .accept(MediaType.TEXT_EVENT_STREAM)
    .retrieve()
    .bodyToFlux(new ParameterizedTypeReference<ServerSentEvent<String>>() {});

events.subscribe(
    event -> log.info("Event: id={}, data={}", event.id(), event.data()),
    error -> log.error("Error: {}", error.getMessage()),
    () -> log.info("Stream completed")
);

// NDJSON 스트리밍
Flux<User> users = webClient.get()
    .uri("/users/stream")
    .accept(MediaType.APPLICATION_NDJSON)
    .retrieve()
    .bodyToFlux(User.class);
```

## 7. 에러 처리

### 7.1 기본 에러 처리

```java
// onStatus로 상태 코드별 처리
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, response ->
        Mono.error(new ClientException("Client error: " + response.statusCode())))
    .onStatus(HttpStatusCode::is5xxServerError, response ->
        Mono.error(new ServerException("Server error: " + response.statusCode())))
    .bodyToMono(User.class);
```

### 7.2 상세 에러 정보 추출

```java
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .onStatus(HttpStatusCode::isError, response ->
        response.bodyToMono(ErrorResponse.class)
            .flatMap(errorBody -> Mono.error(
                new ApiException(response.statusCode(), errorBody.getMessage()))))
    .bodyToMono(User.class);
```

### 7.3 재시도 (Retry)

```java
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class)
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
        .filter(throwable -> throwable instanceof WebClientResponseException.ServiceUnavailable)
        .onRetryExhaustedThrow((spec, signal) ->
            new ServiceUnavailableException("Service unavailable after retries")));
```

### 7.4 Fallback

```java
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class)
    .onErrorResume(WebClientResponseException.NotFound.class,
        e -> Mono.just(User.defaultUser()))
    .onErrorResume(Exception.class,
        e -> Mono.error(new ServiceException("Failed to fetch user", e)));
```

## 8. 타임아웃 설정

### 8.1 전역 타임아웃

```java
HttpClient httpClient = HttpClient.create()
    .responseTimeout(Duration.ofSeconds(10));

WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

### 8.2 요청별 타임아웃

```java
Mono<User> user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class)
    .timeout(Duration.ofSeconds(5))
    .onErrorResume(TimeoutException.class,
        e -> Mono.error(new ServiceTimeoutException("Request timeout")));
```

### 8.3 상세 타임아웃 설정

```java
HttpClient httpClient = HttpClient.create()
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)  // 연결 타임아웃
    .responseTimeout(Duration.ofSeconds(10))            // 응답 타임아웃
    .doOnConnected(conn -> conn
        .addHandlerLast(new ReadTimeoutHandler(10))     // 읽기 타임아웃
        .addHandlerLast(new WriteTimeoutHandler(10)));  // 쓰기 타임아웃

WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

## 9. 인증

### 9.1 Basic 인증

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeaders(headers ->
        headers.setBasicAuth("username", "password"))
    .build();
```

### 9.2 Bearer Token

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + token)
    .build();

// 동적 토큰
@Bean
public WebClient webClient(TokenService tokenService) {
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .filter((request, next) -> tokenService.getToken()
            .map(token -> ClientRequest.from(request)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .build())
            .flatMap(next::exchange))
        .build();
}
```

### 9.3 OAuth2

```java
@Bean
public WebClient webClient(ReactiveOAuth2AuthorizedClientManager manager) {
    ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
        new ServerOAuth2AuthorizedClientExchangeFilterFunction(manager);
    oauth2.setDefaultClientRegistrationId("my-client");

    return WebClient.builder()
        .filter(oauth2)
        .build();
}
```

## 10. 동기 호출 (Blocking)

Spring MVC에서 WebClient를 사용할 때 동기적으로 결과를 얻을 수 있습니다.

```java
// block() 사용 (주의: Reactive 환경에서는 사용 금지)
User user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class)
    .block();

// 타임아웃과 함께 block()
User user = webClient.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class)
    .block(Duration.ofSeconds(5));

// Flux를 List로
List<User> users = webClient.get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(User.class)
    .collectList()
    .block();
```

## 11. 병렬 요청

```java
// 여러 요청 병렬 실행
Mono<User> user1 = webClient.get().uri("/users/1").retrieve().bodyToMono(User.class);
Mono<User> user2 = webClient.get().uri("/users/2").retrieve().bodyToMono(User.class);
Mono<User> user3 = webClient.get().uri("/users/3").retrieve().bodyToMono(User.class);

// Mono.zip으로 결합
Mono<List<User>> users = Mono.zip(user1, user2, user3)
    .map(tuple -> List.of(tuple.getT1(), tuple.getT2(), tuple.getT3()));

// Flux.merge로 결합
Flux<User> users = Flux.merge(user1, user2, user3);

// 동적 리스트
List<Long> ids = List.of(1L, 2L, 3L, 4L, 5L);

Flux<User> users = Flux.fromIterable(ids)
    .flatMap(id -> webClient.get()
        .uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(User.class));
```

## 12. 다음 단계

- [WebSocket](06_WebSocket.md): 실시간 양방향 통신
- [Server-Sent Events](07_Server_Sent_Events.md): 서버 푸시
- [Testing](10_Testing.md): WebClient 테스트
