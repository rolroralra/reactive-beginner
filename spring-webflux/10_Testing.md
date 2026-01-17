# Testing

## 1. 개요

Spring WebFlux 애플리케이션의 테스트는 리액티브 특성을 고려해야 합니다. 이 문서에서는 다양한 테스트 방법을 설명합니다.

## 2. 의존성

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.projectreactor:reactor-test'
}
```

## 3. StepVerifier (Reactor Test)

### 3.1 기본 사용

```java
import reactor.test.StepVerifier;

@Test
void testMono() {
    Mono<String> mono = Mono.just("Hello");

    StepVerifier.create(mono)
        .expectNext("Hello")
        .verifyComplete();
}

@Test
void testFlux() {
    Flux<Integer> flux = Flux.just(1, 2, 3);

    StepVerifier.create(flux)
        .expectNext(1)
        .expectNext(2)
        .expectNext(3)
        .verifyComplete();
}

// expectNextCount 사용
@Test
void testFluxCount() {
    Flux<Integer> flux = Flux.range(1, 100);

    StepVerifier.create(flux)
        .expectNextCount(100)
        .verifyComplete();
}
```

### 3.2 에러 검증

```java
@Test
void testError() {
    Mono<String> mono = Mono.error(new IllegalArgumentException("Invalid input"));

    StepVerifier.create(mono)
        .expectError(IllegalArgumentException.class)
        .verify();
}

@Test
void testErrorMessage() {
    Mono<String> mono = Mono.error(new IllegalArgumentException("Invalid input"));

    StepVerifier.create(mono)
        .expectErrorMessage("Invalid input")
        .verify();
}

@Test
void testErrorMatches() {
    Mono<String> mono = Mono.error(new IllegalArgumentException("Invalid input"));

    StepVerifier.create(mono)
        .expectErrorMatches(e ->
            e instanceof IllegalArgumentException &&
            e.getMessage().contains("Invalid"))
        .verify();
}
```

### 3.3 조건부 검증

```java
@Test
void testWithAssertions() {
    Flux<User> users = userService.findAll();

    StepVerifier.create(users)
        .assertNext(user -> {
            assertThat(user.getName()).isNotEmpty();
            assertThat(user.getEmail()).contains("@");
        })
        .assertNext(user -> assertThat(user.getId()).isPositive())
        .verifyComplete();
}

@Test
void testConsumeWhile() {
    Flux<Integer> flux = Flux.range(1, 10);

    StepVerifier.create(flux)
        .thenConsumeWhile(i -> i <= 10)
        .verifyComplete();
}
```

### 3.4 가상 시간 테스트

```java
@Test
void testWithVirtualTime() {
    // 실제로 1초를 기다리지 않음
    StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofSeconds(1)).take(3))
        .expectSubscription()
        .expectNoEvent(Duration.ofSeconds(1))
        .expectNext(0L)
        .thenAwait(Duration.ofSeconds(1))
        .expectNext(1L)
        .thenAwait(Duration.ofSeconds(1))
        .expectNext(2L)
        .verifyComplete();
}

@Test
void testDelayedMono() {
    StepVerifier.withVirtualTime(() ->
        Mono.just("delayed").delayElement(Duration.ofHours(1)))
        .expectSubscription()
        .expectNoEvent(Duration.ofMinutes(59))
        .thenAwait(Duration.ofMinutes(1))
        .expectNext("delayed")
        .verifyComplete();
}
```

## 4. WebTestClient

### 4.1 기본 설정

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerIntegrationTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldGetUser() {
        webTestClient.get()
            .uri("/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> {
                assertThat(user.getId()).isEqualTo(1L);
                assertThat(user.getName()).isNotEmpty();
            });
    }
}
```

### 4.2 @WebFluxTest (슬라이스 테스트)

```java
@WebFluxTest(UserController.class)
class UserControllerTest {

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
    void shouldReturnNotFound() {
        when(userService.findById(999L)).thenReturn(Mono.empty());

        webTestClient.get()
            .uri("/users/999")
            .exchange()
            .expectStatus().isNotFound();
    }

    @Test
    void shouldGetAllUsers() {
        List<User> users = List.of(
            new User(1L, "홍길동", "hong@example.com"),
            new User(2L, "김철수", "kim@example.com")
        );
        when(userService.findAll()).thenReturn(Flux.fromIterable(users));

        webTestClient.get()
            .uri("/users")
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(User.class)
            .hasSize(2)
            .contains(users.get(0), users.get(1));
    }
}
```

### 4.3 POST 요청 테스트

```java
@Test
void shouldCreateUser() {
    User newUser = new User(null, "홍길동", "hong@example.com");
    User savedUser = new User(1L, "홍길동", "hong@example.com");

    when(userService.save(any(User.class))).thenReturn(Mono.just(savedUser));

    webTestClient.post()
        .uri("/users")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(newUser)
        .exchange()
        .expectStatus().isCreated()
        .expectBody(User.class)
        .value(user -> {
            assertThat(user.getId()).isEqualTo(1L);
            assertThat(user.getName()).isEqualTo("홍길동");
        });
}

@Test
void shouldValidateUser() {
    User invalidUser = new User(null, "", "invalid-email");

    webTestClient.post()
        .uri("/users")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(invalidUser)
        .exchange()
        .expectStatus().isBadRequest();
}
```

### 4.4 헤더 및 쿠키 테스트

```java
@Test
void shouldIncludeAuthHeader() {
    when(userService.findById(1L)).thenReturn(Mono.just(user));

    webTestClient.get()
        .uri("/users/1")
        .header("Authorization", "Bearer token123")
        .exchange()
        .expectStatus().isOk();
}

@Test
void shouldSetCookie() {
    webTestClient.post()
        .uri("/auth/login")
        .bodyValue(credentials)
        .exchange()
        .expectStatus().isOk()
        .expectCookie().exists("sessionId")
        .expectCookie().httpOnly("sessionId", true);
}

@Test
void shouldReturnHeaders() {
    webTestClient.get()
        .uri("/users/1")
        .exchange()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectHeader().exists("X-Request-Id");
}
```

### 4.5 JSON Path 검증

```java
@Test
void shouldValidateJsonResponse() {
    when(userService.findById(1L)).thenReturn(Mono.just(user));

    webTestClient.get()
        .uri("/users/1")
        .exchange()
        .expectStatus().isOk()
        .expectBody()
        .jsonPath("$.id").isEqualTo(1)
        .jsonPath("$.name").isEqualTo("홍길동")
        .jsonPath("$.email").exists()
        .jsonPath("$.password").doesNotExist();
}

@Test
void shouldValidateListResponse() {
    webTestClient.get()
        .uri("/users")
        .exchange()
        .expectStatus().isOk()
        .expectBody()
        .jsonPath("$.length()").isEqualTo(2)
        .jsonPath("$[0].name").isEqualTo("홍길동")
        .jsonPath("$[*].email").value(emails -> {
            assertThat((List<String>) emails).allMatch(e -> e.contains("@"));
        });
}
```

## 5. Functional Endpoints 테스트

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
}
```

## 6. Service 레이어 테스트

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldFindUserById() {
        User user = new User(1L, "홍길동", "hong@example.com");
        when(userRepository.findById(1L)).thenReturn(Mono.just(user));

        StepVerifier.create(userService.findById(1L))
            .expectNext(user)
            .verifyComplete();
    }

    @Test
    void shouldReturnEmptyWhenUserNotFound() {
        when(userRepository.findById(999L)).thenReturn(Mono.empty());

        StepVerifier.create(userService.findById(999L))
            .verifyComplete();
    }

    @Test
    void shouldSaveUser() {
        User user = new User(null, "홍길동", "hong@example.com");
        User savedUser = new User(1L, "홍길동", "hong@example.com");

        when(userRepository.save(user)).thenReturn(Mono.just(savedUser));

        StepVerifier.create(userService.save(user))
            .assertNext(result -> {
                assertThat(result.getId()).isEqualTo(1L);
                assertThat(result.getName()).isEqualTo("홍길동");
            })
            .verifyComplete();
    }
}
```

## 7. WebClient 테스트

### 7.1 MockWebServer 사용

```groovy
testImplementation 'com.squareup.okhttp3:mockwebserver'
```

```java
class ExternalApiServiceTest {

    private MockWebServer mockWebServer;
    private ExternalApiService service;

    @BeforeEach
    void setUp() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();

        WebClient webClient = WebClient.builder()
            .baseUrl(mockWebServer.url("/").toString())
            .build();

        service = new ExternalApiService(webClient);
    }

    @AfterEach
    void tearDown() throws IOException {
        mockWebServer.shutdown();
    }

    @Test
    void shouldFetchData() {
        mockWebServer.enqueue(new MockResponse()
            .setBody("{\"id\": 1, \"name\": \"Test\"}")
            .setHeader("Content-Type", "application/json"));

        StepVerifier.create(service.fetchData(1))
            .assertNext(data -> {
                assertThat(data.getId()).isEqualTo(1);
                assertThat(data.getName()).isEqualTo("Test");
            })
            .verifyComplete();

        RecordedRequest request = mockWebServer.takeRequest();
        assertThat(request.getPath()).isEqualTo("/data/1");
    }

    @Test
    void shouldHandleError() {
        mockWebServer.enqueue(new MockResponse()
            .setResponseCode(500)
            .setBody("{\"error\": \"Server Error\"}"));

        StepVerifier.create(service.fetchData(1))
            .expectError(ServiceException.class)
            .verify();
    }
}
```

### 7.2 WireMock 사용

```groovy
testImplementation 'org.wiremock:wiremock:3.3.1'
```

```java
@WireMockTest(httpPort = 8089)
class ExternalApiServiceWireMockTest {

    private ExternalApiService service;

    @BeforeEach
    void setUp() {
        WebClient webClient = WebClient.builder()
            .baseUrl("http://localhost:8089")
            .build();
        service = new ExternalApiService(webClient);
    }

    @Test
    void shouldFetchData() {
        stubFor(get(urlEqualTo("/data/1"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"id\": 1, \"name\": \"Test\"}")));

        StepVerifier.create(service.fetchData(1))
            .assertNext(data -> assertThat(data.getName()).isEqualTo("Test"))
            .verifyComplete();
    }
}
```

## 8. SSE 테스트

```java
@Test
void shouldStreamEvents() {
    webTestClient.get()
        .uri("/sse/events")
        .accept(MediaType.TEXT_EVENT_STREAM)
        .exchange()
        .expectStatus().isOk()
        .returnResult(String.class)
        .getResponseBody()
        .take(3)
        .as(StepVerifier::create)
        .expectNextCount(3)
        .verifyComplete();
}
```

## 9. WebSocket 테스트

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class WebSocketTest {

    @LocalServerPort
    private int port;

    @Test
    void shouldEchoMessages() {
        WebSocketClient client = new ReactorNettyWebSocketClient();
        URI uri = URI.create("ws://localhost:" + port + "/ws/echo");

        List<String> received = new CopyOnWriteArrayList<>();

        client.execute(uri, session -> {
            Flux<WebSocketMessage> output = Flux.just("Hello", "World")
                .map(session::textMessage);

            return session.send(output)
                .thenMany(session.receive()
                    .map(WebSocketMessage::getPayloadAsText)
                    .doOnNext(received::add)
                    .take(2))
                .then();
        }).block(Duration.ofSeconds(5));

        assertThat(received).containsExactly("Echo: Hello", "Echo: World");
    }
}
```

## 10. 테스트 유틸리티

### 10.1 PublisherProbe

```java
@Test
void testWithProbe() {
    PublisherProbe<Void> probe = PublisherProbe.empty();

    Mono<String> result = someService.process()
        .then(probe.mono())
        .thenReturn("done");

    StepVerifier.create(result)
        .expectNext("done")
        .verifyComplete();

    probe.assertWasSubscribed();
    probe.assertWasRequested();
    probe.assertWasNotCancelled();
}
```

### 10.2 TestPublisher

```java
@Test
void testWithTestPublisher() {
    TestPublisher<String> testPublisher = TestPublisher.create();

    Flux<String> flux = testPublisher.flux();

    StepVerifier.create(flux)
        .then(() -> testPublisher.emit("one", "two", "three"))
        .expectNext("one", "two", "three")
        .verifyComplete();
}
```

## 11. 다음 단계

- [Spring Security 통합](11_Spring_Security_통합.md): 보안 테스트
- [R2DBC](12_R2DBC.md): 데이터베이스 테스트
- [예외 처리](08_예외_처리.md): 에러 시나리오 테스트
