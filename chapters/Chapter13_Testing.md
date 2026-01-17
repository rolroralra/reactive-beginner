# Chapter13. Testing

---

## StepVerifier를 사용한 테스팅

> Reactor에서 가장 일반적인 테스트 방식은
> Flux 또는 Mono를 Reactor Sequence로 정의한 후,
> 구독 시점에 해당 Operator 체인이 시나리오대로 동작하는지를 테스트하는 것입니다.

- Reactor Sequence에서 발생하는 Signal 이벤트를 테스트하는 것

---

## Signal 이벤트 테스트

```java
@Test
public void sayHelloReactorTest() {
    StepVerifier
            .create(Mono.just("Hello Reactor")) // 테스트 대상 Sequence 생성
            .expectNext("Hello Reactor")    // emit 된 데이터 검증
            .expectComplete()   // onComplete Signal 검증
            .verify();          // 검증 실행.
}
```

### expect 메서드 목록

| 메서드 | 설명 |
|--------|------|
| `expectSubscription()` | 구독이 이루어짐을 기대 |
| `expectNext(T t)` | onNext Signal을 통해 전달되는 값이 파라미터로 전달된 값과 같음을 기대 |
| `expectComplete()` | onComplete Signal이 전송되기를 기대 |
| `expectError()` | onError Signal이 전송되기를 기대 |
| `expectNextCount(long count)` | 구독 시점 또는 이전 expectNext()를 통해 기댓값이 평가된 데이터 이후부터 emit된 수를 기대 |
| `expectNoEvent(Duration duration)` | 주어진 시간 동안 Signal 이벤트가 발생하지 않았음을 기대 |
| `expectAccessibleContext()` | 구독 시점 이후에 Context가 전파되었음을 기대 |
| `expectNextSequence(Iterable<T> iterable)` | emit된 데이터들이 파라미터로 전달된 Iterable의 요소와 매치됨을 기대 |

### verify 메서드 목록

| 메서드 | 설명 |
|--------|------|
| `verify()` | 검증을 트리거 |
| `verifyComplete()` | 검증을 트리거하고, onComplete Signal을 기대 |
| `verifyError()` | 검증을 트리거하고, onError Signal을 기대 |
| `verifyTimeout(Duration duration)` | 검증을 트리거하고, 주어진 시간이 초과되어도 Publisher가 종료되지 않음을 기대 |

### Test Description 넣기

- `as()`
  - 이전 기댓값 평가 단계에 대한 설명(description)을 추가할 수 있다.
- `StepVerifierOptions.create().scenarioName()`
  - 테스트에 대한 설명(description)을 추가할 수 있다.

---

## Time-based 테스트

- StepVerifier는 가상의 시간을 이용해 미래에 실행되는 Reactor Sequence의 시간을 앞당겨 테스트할 수 있는 기능을 지원한다.
  - 이를 통해 실제 시간이 오래 걸리는 테스트를 빠르게 수행할 수 있다.

```java
StepVerifier
        .withVirtualTime(() -> TimeBasedTestExample.getCOVID19Count(
                        Flux.interval(Duration.ofHours(1)).take(1)
                )
        )
        .expectSubscription()
        .then(() -> VirtualTimeScheduler
                            .get()
                            .advanceTimeBy(Duration.ofHours(1)))
        .expectNextCount(11)
        .expectComplete()
        .verify();
```

- `withVirtualTime()`
  - VirtualTimeScheduler를 사용하여 가상의 시간을 흐르게 할 수 있다.
- `VirtualTimeScheduler.get().advanceTimeBy()`
  - 주어진 시간만큼 가상의 시간을 앞당긴다.

```java
StepVerifier
        .create(TimeBasedTestExample.getCOVID19Count(
                        Flux.interval(Duration.ofMinutes(1)).take(1)
                )
        )
        .expectSubscription()
        .expectNextCount(11)
        .expectComplete()
        .verify(Duration.ofSeconds(3));

/*
java.lang.AssertionError: VerifySubscriber timed out on reactor.core.publisher.FluxFlatMap$FlatMapMain@564928fc
*/
```

- `verify(Duration duration)`
  - 주어진 시간 내에 Sequence가 완료되지 않으면 Timeout을 발생시킨다.

```java
StepVerifier
        .withVirtualTime(() -> TimeBasedTestExample.getVoteCount(
                        Flux.interval(Duration.ofMinutes(1))
                )
        )
        .expectSubscription()
        .expectNoEvent(Duration.ofMinutes(1))
        .expectNoEvent(Duration.ofMinutes(1))
        .expectNoEvent(Duration.ofMinutes(1))
        .expectNoEvent(Duration.ofMinutes(1))
        .expectNoEvent(Duration.ofMinutes(1))
        .expectNextCount(5)
        .expectComplete()
        .verify();
```

- `expectNoEvent(Duration duration)`
  - 주어진 시간 동안 어떤 이벤트도 발생하지 않음을 기대한다.
  - 내부적으로 시간을 앞당긴다.

---

## Backpressure 테스트

- Backpressure에 대한 테스트 역시 수행 가능
  - Subscriber가 Publisher로부터 적절한 양의 데이터를 요청하는지 테스트

```java
StepVerifier
        .create(BackpressureTestExample.generateNumber(), 1L)
        .thenConsumeWhile(num -> num >= 1)
        .expectError()
        .verifyThenAssertThat()
        .hasDroppedElements();
```

- `thenConsumeWhile()` 메서드를 사용하여 emit되는 데이터를 소비하도록 설정
- `verifyThenAssertThat()` 메서드를 사용하면 검증을 트리거한 후, 추가적인 Assertion을 할 수 있다.
  - `hasDroppedElements()`, `hasDropped()`, `hasDroppedErrors()` 등

---

## Context 테스트

- Context에 대한 테스트 역시 수행 가능
  - Context가 올바르게 전파되었는지 테스트
- Context 테스트 이후, `then()` 을 호출해서 Signal 이벤트에 대한 평가를 진행할 수 있도록 한다.

```java
@Test
public void getSecretMessageTest() {
    Mono<String> source = Mono.just("hello");

    StepVerifier
            .create(
                ContextTestExample
                    .getSecretMessage(source)
                    .contextWrite(context ->
                                    context.put("secretMessage", "Hello, Reactor"))
                    .contextWrite(context -> context.put("secretKey", "aGVsbG8="))
            )
            .expectSubscription()
            .expectAccessibleContext()
            .hasKey("secretKey")
            .hasKey("secretMessage")
            .then()
            .expectNext("Hello, Reactor")
            .expectComplete()
            .verify();
}

public static Mono<String> getSecretMessage(Mono<String> keySource) {
    return keySource
            .zipWith(Mono.deferContextual(ctx ->
                                           Mono.just((String)ctx.get("secretKey"))))
            .filter(tp ->
                        tp.getT1().equals(
                               new String(Base64Utils.decodeFromString(tp.getT2())))
            )
            .transformDeferredContextual(
                    (mono, ctx) -> mono.map(notUse -> ctx.get("secretMessage"))
            );
}
```

---

## Record 테스트

- `recordWith()`
  - emit된 데이터를 기록하는 용도로 사용
- `thenConsumeWhile()`
  - 파라미터로 전달한 Predicate와 일치하는 데이터는 다음 단계에서 소비
- `consumeRecordedWith()`
  - 컬렉션에 기록된 데이터를 소비
- `expectRecordedMatches()`
  - 컬렉션에 기록된 데이터가 Predicate와 일치하는지 검증

```java
StepVerifier
        .create(RecordTestExample.getCapitalizedCountry(
                Flux.just("korea", "england", "canada", "india")))
        .expectSubscription()
        .recordWith(ArrayList::new)
        .thenConsumeWhile(country -> !country.isEmpty())
        .consumeRecordedWith(countries -> {
            assertThat(
                    countries
                            .stream()
                            .allMatch(country ->
                                    Character.isUpperCase(country.charAt(0))),
                    is(true)
            );
        })
        .expectComplete()
        .verify();
```

```java
StepVerifier
        .create(RecordTestExample.getCapitalizedCountry(
                Flux.just("korea", "england", "canada", "india")))
        .expectSubscription()
        .recordWith(ArrayList::new)
        .thenConsumeWhile(country -> !country.isEmpty())
        .expectRecordedMatches(countries ->
                countries
                        .stream()
                        .allMatch(country ->
                                Character.isUpperCase(country.charAt(0))))
        .expectComplete()
        .verify();
```

---

## TestPublisher를 사용한 테스팅

- `TestPublisher` 를 사용하면 개발자가 직접 프로그래밍 방식으로 Signal을 발생시키면서 원하는 상황을 미세하게 재연하며 테스트를 진행할 수 있다.
- **TestPublisher가 발생시키는 Signal 종류**
  - `next(T)` or `next(T, T, ...)`: 1개 이상의 onNext Signal 발생
  - `emit(T...)`: 1개 이상의 onNext Signal을 발생시킨 후, onComplete Signal 발생
  - `complete()`: onComplete Signal 발생
  - `error(Throwable)`: onError Signal 발생

```java
TestPublisher<Integer> source = TestPublisher.create();

StepVerifier
        .create(GeneralTestExample.divideByTwo(source.flux()))
        .expectSubscription()
        .then(() -> source.emit(2, 4, 6, 8, 10))
        .expectNext(1, 2, 3, 4)
        .expectError()
        .verify();
```

### Misbehaving TestPublisher

- `TestPublisher.Violation`
  - 리액티브 스트림즈 사양 위반을 허용하는 TestPublisher
  - `ALLOW_NULL`: null 전송 허용
  - `CLEANUP_ON_TERMINATE`: 종료 후 cleanup 동작 허용
  - `DEFER_CANCELLATION`: 취소 Signal을 무시
  - `REQUEST_OVERFLOW`: 요청 오버플로우 허용

```java
TestPublisher<Integer> source =
        TestPublisher.createNoncompliant(TestPublisher.Violation.ALLOW_NULL);

StepVerifier
        .create(GeneralTestExample.divideByTwo(source.flux()))
        .expectSubscription()
        .then(() -> {
            Arrays.asList(2, 4, 6, 8, null).stream()
                    .forEach(data -> source.next(data));
            source.complete();
        })
        .expectNext(1, 2, 3, 4, 5)
        .expectComplete()
        .verify();
```

---

## PublisherProbe를 사용한 테스팅

- `PublisherProbe` 를 사용하면 Sequence의 실행이 분기되는 상황에서 Publisher가 어느 경로로 실행되는지 테스트할 수 있다.
- `PublisherProbe.of`
  - 테스트하고자 하는 Publisher를 PublisherProbe로 래핑
- **PublisherProbe 가 제공하는 Assertion**
  - `assertWasSubscribed()`: 구독이 이루어졌음을 검증
  - `assertWasRequested()`: 요청이 이루어졌음을 검증
  - `assertWasNotCancelled()`: 취소가 이루어지지 않았음을 검증

```java
@Test
public void publisherProbeTest() {
    PublisherProbe<String> probe =
            PublisherProbe.of(PublisherProbeTestExample.supplyStandbyPower());

    StepVerifier
            .create(PublisherProbeTestExample
                    .processTask(
                            PublisherProbeTestExample.supplyMainPower(),
                            probe.mono())
            )
            .expectNextCount(1)
            .verifyComplete();

    probe.assertWasSubscribed();
    probe.assertWasRequested();
    probe.assertWasNotCancelled();
}

public static Mono<String> processTask(Mono<String> main, Mono<String> standby) {
    return main
            .flatMap(Mono::just)
            .switchIfEmpty(standby);
}

public static Mono<String> supplyMainPower() {
    return Mono.empty();
}

public static Mono supplyStandbyPower() {
    return Mono.just("# supply Standby Power");
}
```
