# Chapter12. Debugging

---

## Reactor에서의 디버깅 방법

- 동기식 또는 명령형 프로그래밍 방식은 Exception이 발생 했을 때
  - Stacktrace를 통해 에러가 발생한 지점을 비교적 정확하게 파악할 수 있다.
- `Hooks.onOperatorDebug()` 를 통해서 Reactor에서의 디버그 모드 활성화 가능
- `ReactorDebugAgent`를 사용하여 프로덕션 환경에서 디버그 모드 활성화 가능

### 1. Hooks.onOperatorDebug()

- Application 내에 있는 모든 Operator의 Stacktrace를 캡처한다.
- Error가 발생하면 캡처한 정보를 기반으로 Error가 발생한 Assembly의 Stacktrace를 원본 Stacktrace 중간에 끼워 넣는다.

```java
public static Map<String, String> fruits = new HashMap<>();

static {
    fruits.put("banana", "바나나");
    fruits.put("apple", "사과");
    fruits.put("pear", "배");
    fruits.put("grape", "포도");
}

public static void main(String[] args) throws InterruptedException {
    Hooks.onOperatorDebug();

    Flux
        .fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
        .subscribeOn(Schedulers.boundedElastic())
        .publishOn(Schedulers.parallel())
        .map(String::toLowerCase)
        .map(fruit -> fruit.substring(0, fruit.length() - 1))
        .map(fruits::get)
        .map(translated -> "맛있는 " + translated)
        .subscribe(
                log::info,
                error -> log.error("# onError:", error));

    Thread.sleep(100L);
}

/*
21:35:36.964 [parallel-1] INFO - 맛있는 바나나
21:35:36.965 [parallel-1] INFO - 맛있는 사과
21:35:36.966 [parallel-1] INFO - 맛있는 배
21:35:36.981 [parallel-1] ERROR- # onError:
java.lang.NullPointerException: The mapper [...] returned a null value.
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(...)
    Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException:
Assembly trace from producer [reactor.core.publisher.FluxMapFuseable] :
    reactor.core.publisher.Flux.map(Flux.java:6271)
    chapter12.Example12_1.main(Example12_1.java:35)
Error has been observed at the following site(s):
    *__Flux.map ⇢ at chapter12.Example12_1.main(Example12_1.java:35)
    |_ Flux.map ⇢ at chapter12.Example12_1.main(Example12_1.java:36)
Original Stack Trace:
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(...)
    ...
*/
```

### 2. ReactorDebugAgent

- Reactor Tool에서 지원하는 `ReactorDebugAgent`를 사용하여 프로덕션 환경에서 디버그 모드를 대체할 수 있다.

#### Assembly

- Operator에서 리턴하는 새로운 Mono 또는 Flux가 선언된 지점을 Assembly라고 한다.

#### Traceback

- 디버그 모드를 활성화하면 Operator의 Assembly정보를 캡처하는데 이중에서 에러가 발생한 Operator의 Stacktrace를 캡처한 Assembly 정보를 Traceback이라고 한다.
- Traceback은 Suppressed Exception 형태로 원본 Stacktrace에 추가된다.

#### Production 환경에서의 디버깅 설정

- Reactor에서는 Application 내 모든 Operator 체인의 Stacktrace 캡처 비용을 지불하지 않고 디버깅 정보를 추가할 수 있도록 별도의 Java Agent를 제공합니다.
- 아래와 같이 2가지를 설정하면, `ReactorDebugAgent` 를 활성화할 수 있다.

```kotlin
implementation("io.projectreactor:reactor-tools:3.5.8")
```

```yaml
spring.reactor.debug-agent.enabled: true
```

- `ReactorDebugAgent.init()`
  - Application 내의 클래스를 변환하고 디버그 추적 정보를 삽입하는 작업을 수행

---

## checkpoint() Operator를 사용한 디버깅

> Hooks.onOperatorDebug() 을 통한 디버그 모드를 활성화하는 방법이 Application 내에 있는 모든 Operator에서 Stacktrace를 캡처하는 반면에,

**`checkpoint()` Operator를 사용하면 특정 Operator 체인 내의 Stacktrace만 캡처한다.**

### 1. Traceback을 출력하는 방법

- `checkpoint()` 를 사용하면 실제 에러가 발생한 assembly 지점 또는 에러가 전파된 assembly 지점의 traceback이 추가된다.

```java
Flux
    .just(2, 4, 6, 8)
    .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
    .map(num -> num + 2)
    .checkpoint()
    .subscribe(
            data -> log.info("# onNext: {}", data),
            error -> log.error("# onError:", error)
    );

/*
...
Error has been observed at the following site(s):
    *__checkpoint() ⇢ at chapter12.Example12_2.main(Example12_2.java:17)
...
*/
```

```java
Flux
    .just(2, 4, 6, 8)
    .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
    .checkpoint()
    .map(num -> num + 2)
    .checkpoint()
    .subscribe(
            data -> log.info("# onNext: {}", data),
            error -> log.error("# onError:", error)
    );

/*
...
Error has been observed at the following site(s):
    *__checkpoint() ⇢ at chapter12.Example12_3.main(Example12_3.java:16)
    |_ checkpoint() ⇢ at chapter12.Example12_3.main(Example12_3.java:18)
...
*/
```

### 2. Traceback 출력 없이 식별자를 포함한 Description을 출력해서 에러 발생 지점을 예상하는 방법

- `checkpoint(description)` 을 사용하면 에러 발생 시 Traceback을 생략하고 description을 통해 에러 발생 지점을 예상할 수 있다.

```java
Flux
    .just(2, 4, 6, 8)
    .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
    .checkpoint("Example12_4.zipWith.checkpoint")
    .map(num -> num + 2)
    .checkpoint("Example12_4.map.checkpoint")
    .subscribe(
            data -> log.info("# onNext: {}", data),
            error -> log.error("# onError:", error)
    );

/*
...
Error has been observed at the following site(s):
    *__checkpoint ⇢ Example12_4.zipWith.checkpoint
    |_ checkpoint ⇢ Example12_4.map.checkpoint
...
*/
```

### 3. Traceback과 Description을 모두 출력하는 방법

- `checkpoint(description, forceStackTrace)` 를 사용하면 description과 Traceback을 모두 출력할 수 있다.

```java
Flux
    .just(2, 4, 6, 8)
    .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
    .checkpoint("Example12_4.zipWith.checkpoint", true)
    .map(num -> num + 2)
    .checkpoint("Example12_4.map.checkpoint", true)
    .subscribe(
            data -> log.info("# onNext: {}", data),
            error -> log.error("# onError:", error)
    );

/*
...
Error has been observed at the following site(s):
    *__checkpoint(Example12_4.zipWith.checkpoint) ⇢ at chapter12.Example12_5.main(Example12_5.java:17)
    |_     checkpoint(Example12_4.map.checkpoint) ⇢ at chapter12.Example12_5.main(Example12_5.java:19)
...
*/
```

### 4. log() Operator를 사용한 디버깅

- `log()` Operator는 Reactor Sequence의 동작을 로그로 출력한다.

```java
Map<String, String> fruits = Map.of(
        "banana", "바나나",
        "apple", "사과",
        "pear", "배",
        "grape", "포도"
);

Flux.fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
        .map(String::toLowerCase)
        .map(fruit -> fruit.substring(0, fruit.length() - 1))
        .log()
//      .log("Fruit.Substring", Level.FINE)
        .map(fruits::get)
        .subscribe(
                log::info,
                error -> log.error("# onError:", error));

/*
00:16:20.671 [main] DEBUG- Using Slf4j logging framework
00:16:20.679 [main] INFO - | onSubscribe([Fuseable] FluxMapFuseable.MapFuseableSubscriber)
00:16:20.680 [main] INFO - | request(unbounded)
00:16:20.680 [main] INFO - | onNext(banana)
00:16:20.680 [main] INFO - 바나나
00:16:20.681 [main] INFO - | onNext(apple)
00:16:20.681 [main] INFO - 사과
00:16:20.681 [main] INFO - | onNext(pear)
00:16:20.681 [main] INFO - 배
00:16:20.681 [main] INFO - | onNext(melon)
00:16:20.683 [main] INFO - | cancel()
00:16:20.684 [main] ERROR- # onError:
...
*/
```
