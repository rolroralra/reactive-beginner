# Annotated Controllers

## 1. 개요

Spring WebFlux의 Annotated Controllers는 Spring MVC와 동일한 어노테이션을 사용합니다. 기존 Spring MVC 개발자가 쉽게 적응할 수 있는 방식입니다.

```java
@RestController
@RequestMapping("/api")
public class MyController {

    @GetMapping("/hello")
    public Mono<String> hello() {
        return Mono.just("Hello!");
    }
}
```

## 2. 지원 어노테이션

### 2.1 클래스 레벨

| 어노테이션 | 설명 |
|-----------|------|
| `@Controller` | 컨트롤러 클래스 선언 |
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@RequestMapping` | 기본 URL 매핑 |

### 2.2 메서드 레벨

| 어노테이션 | HTTP 메서드 |
|-----------|------------|
| `@GetMapping` | GET |
| `@PostMapping` | POST |
| `@PutMapping` | PUT |
| `@DeleteMapping` | DELETE |
| `@PatchMapping` | PATCH |

## 3. 요청 매핑

### 3.1 URL 패턴

```java
@RestController
@RequestMapping("/users")
public class UserController {

    // GET /users
    @GetMapping
    public Flux<User> list() { ... }

    // GET /users/123
    @GetMapping("/{id}")
    public Mono<User> get(@PathVariable Long id) { ... }

    // GET /users/search?name=홍길동
    @GetMapping("/search")
    public Flux<User> search(@RequestParam String name) { ... }

    // GET /users/123/orders/456
    @GetMapping("/{userId}/orders/{orderId}")
    public Mono<Order> getOrder(
            @PathVariable Long userId,
            @PathVariable Long orderId) { ... }
}
```

### 3.2 Ant 스타일 패턴

```java
@GetMapping("/files/**")           // /files/a/b/c 매칭
@GetMapping("/docs/{name:[a-z]+}") // 정규식 패턴
@GetMapping("/items/{id:\\d+}")    // 숫자만 매칭
```

### 3.3 Content-Type 및 Accept 매핑

```java
// Content-Type: application/json 요청만 처리
@PostMapping(value = "/users", consumes = MediaType.APPLICATION_JSON_VALUE)
public Mono<User> create(@RequestBody User user) { ... }

// Accept: application/json 응답
@GetMapping(value = "/users", produces = MediaType.APPLICATION_JSON_VALUE)
public Flux<User> list() { ... }

// 여러 Content-Type 지원
@PostMapping(
    value = "/upload",
    consumes = {MediaType.MULTIPART_FORM_DATA_VALUE, MediaType.APPLICATION_JSON_VALUE}
)
public Mono<String> upload(...) { ... }
```

## 4. 요청 파라미터

### 4.1 @PathVariable

```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// 변수명이 다른 경우
@GetMapping("/users/{userId}")
public Mono<User> getUser(@PathVariable("userId") Long id) {
    return userService.findById(id);
}

// 선택적 PathVariable
@GetMapping({"/users", "/users/{id}"})
public Mono<Object> getUsers(@PathVariable(required = false) Long id) {
    if (id != null) {
        return userService.findById(id).cast(Object.class);
    }
    return userService.findAll().collectList().cast(Object.class);
}
```

### 4.2 @RequestParam

```java
// 필수 파라미터
@GetMapping("/search")
public Flux<User> search(@RequestParam String keyword) { ... }

// 선택적 파라미터 (기본값)
@GetMapping("/list")
public Flux<User> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    return userService.findAll()
        .skip((long) page * size)
        .take(size);
}

// 선택적 파라미터 (Optional)
@GetMapping("/search")
public Flux<User> search(
        @RequestParam Optional<String> name,
        @RequestParam Optional<String> email) {
    // ...
}

// 다중 값
@GetMapping("/filter")
public Flux<User> filter(@RequestParam List<String> tags) { ... }
```

### 4.3 @RequestHeader

```java
@GetMapping("/info")
public Mono<String> getInfo(
        @RequestHeader("Authorization") String auth,
        @RequestHeader(value = "X-Custom-Header", required = false) String custom) {
    return Mono.just("Auth: " + auth);
}

// 모든 헤더
@GetMapping("/headers")
public Mono<Map<String, String>> getAllHeaders(@RequestHeader Map<String, String> headers) {
    return Mono.just(headers);
}
```

### 4.4 @CookieValue

```java
@GetMapping("/welcome")
public Mono<String> welcome(
        @CookieValue(value = "sessionId", required = false) String sessionId) {
    if (sessionId != null) {
        return Mono.just("Welcome back!");
    }
    return Mono.just("Hello, new user!");
}
```

### 4.5 @RequestBody

```java
@PostMapping("/users")
public Mono<User> createUser(@RequestBody User user) {
    return userService.save(user);
}

// Mono로 받기 (스트리밍)
@PostMapping("/users")
public Mono<User> createUser(@RequestBody Mono<User> userMono) {
    return userMono.flatMap(userService::save);
}

// Flux로 받기 (스트리밍)
@PostMapping("/users/batch")
public Flux<User> createUsers(@RequestBody Flux<User> usersFlux) {
    return usersFlux.flatMap(userService::save);
}
```

## 5. 응답 처리

### 5.1 기본 응답

```java
// Mono 반환 (단일 값)
@GetMapping("/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// Flux 반환 (다중 값)
@GetMapping
public Flux<User> getAllUsers() {
    return userService.findAll();
}

// void 반환
@DeleteMapping("/{id}")
public Mono<Void> deleteUser(@PathVariable Long id) {
    return userService.deleteById(id);
}
```

### 5.2 ResponseEntity

```java
@GetMapping("/{id}")
public Mono<ResponseEntity<User>> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(user -> ResponseEntity.ok(user))
        .defaultIfEmpty(ResponseEntity.notFound().build());
}

// 헤더 추가
@PostMapping
public Mono<ResponseEntity<User>> createUser(@RequestBody User user) {
    return userService.save(user)
        .map(savedUser -> ResponseEntity
            .created(URI.create("/users/" + savedUser.getId()))
            .header("X-Custom-Header", "value")
            .body(savedUser));
}

// 상태 코드 지정
@PostMapping
public Mono<ResponseEntity<User>> createUser(@RequestBody User user) {
    return userService.save(user)
        .map(savedUser -> ResponseEntity
            .status(HttpStatus.CREATED)
            .body(savedUser));
}
```

### 5.3 @ResponseStatus

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public Mono<User> createUser(@RequestBody User user) {
    return userService.save(user);
}

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public Mono<Void> deleteUser(@PathVariable Long id) {
    return userService.deleteById(id);
}
```

## 6. 스트리밍 응답

### 6.1 Server-Sent Events

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(seq -> ServerSentEvent.<String>builder()
            .id(String.valueOf(seq))
            .event("message")
            .data("Event " + seq)
            .build());
}
```

### 6.2 NDJSON (Newline Delimited JSON)

```java
@GetMapping(value = "/users/stream", produces = MediaType.APPLICATION_NDJSON_VALUE)
public Flux<User> streamUsers() {
    return userService.findAll()
        .delayElements(Duration.ofMillis(100));
}
```

## 7. 유효성 검사

### 7.1 의존성 추가

```groovy
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

### 7.2 DTO에 검증 어노테이션 추가

```java
public class CreateUserRequest {

    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 50, message = "이름은 2~50자 사이여야 합니다")
    private String name;

    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "유효한 이메일 형식이 아닙니다")
    private String email;

    @Min(value = 0, message = "나이는 0 이상이어야 합니다")
    @Max(value = 150, message = "나이는 150 이하여야 합니다")
    private Integer age;

    // Getters and Setters
}
```

### 7.3 Controller에서 검증

```java
@PostMapping
public Mono<User> createUser(@Valid @RequestBody CreateUserRequest request) {
    return userService.save(request.toUser());
}

// 또는 Mono로 받는 경우
@PostMapping
public Mono<User> createUser(@RequestBody Mono<CreateUserRequest> requestMono) {
    return requestMono
        .doOnNext(this::validate)
        .map(CreateUserRequest::toUser)
        .flatMap(userService::save);
}

private void validate(CreateUserRequest request) {
    Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
    Set<ConstraintViolation<CreateUserRequest>> violations = validator.validate(request);
    if (!violations.isEmpty()) {
        throw new ConstraintViolationException(violations);
    }
}
```

## 8. 파일 업로드/다운로드

### 8.1 파일 업로드

```java
@PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public Mono<String> uploadFile(@RequestPart("file") FilePart filePart) {
    Path destination = Path.of("uploads", filePart.filename());
    return filePart.transferTo(destination)
        .then(Mono.just("Uploaded: " + filePart.filename()));
}

// 여러 파일 업로드
@PostMapping(value = "/upload/multiple", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public Flux<String> uploadFiles(@RequestPart("files") Flux<FilePart> filePartFlux) {
    return filePartFlux.flatMap(filePart -> {
        Path destination = Path.of("uploads", filePart.filename());
        return filePart.transferTo(destination)
            .then(Mono.just("Uploaded: " + filePart.filename()));
    });
}
```

### 8.2 파일 다운로드

```java
@GetMapping("/download/{filename}")
public Mono<ResponseEntity<Resource>> downloadFile(@PathVariable String filename) {
    Path filePath = Path.of("uploads", filename);
    Resource resource = new FileSystemResource(filePath);

    return Mono.just(ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + filename + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(resource));
}
```

## 9. 모델 바인딩

### 9.1 @ModelAttribute

```java
// Form 데이터 바인딩
@PostMapping("/register")
public Mono<String> register(@ModelAttribute UserForm form) {
    return userService.register(form)
        .map(user -> "redirect:/users/" + user.getId());
}

// 공통 모델 속성
@ModelAttribute("currentUser")
public Mono<User> currentUser(ServerWebExchange exchange) {
    return exchange.getPrincipal()
        .cast(Authentication.class)
        .map(auth -> (User) auth.getPrincipal());
}
```

## 10. CORS 설정

### 10.1 어노테이션 방식

```java
@RestController
@CrossOrigin(origins = "http://localhost:3000")
@RequestMapping("/api/users")
public class UserController {

    @CrossOrigin(origins = "*", maxAge = 3600)
    @GetMapping
    public Flux<User> list() { ... }
}
```

### 10.2 전역 설정

```java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

## 11. 다음 단계

- [Functional Endpoints](04_Functional_Endpoints.md): 함수형 엔드포인트 가이드
- [예외 처리](08_예외_처리.md): 에러 핸들링 방법
- [WebClient](05_WebClient.md): 리액티브 HTTP 클라이언트
