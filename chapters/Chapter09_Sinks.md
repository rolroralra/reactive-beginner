# Chapter09. Sinks

---

## Sinks란?

> 리액티브 스트림즈의 Signal을 프로그래밍 방식으로 푸시할 수 있는 구조

- Flux 또는 Mono의 의미 체계를 가지는 리액티브 스트림즈 Signal을 프로그래밍 방식으로 푸시할 수 있는 구조

---

## Reactor에서 프로그래밍 방식으로 Signal을 전송하는 가장 일반적인 방법

- `generate()` Operator
- `create()` Operator
- Single Thread 기반
- 따라서 스레드 안정성 보장 X

### 코드 예시

#### create() Operator를 통한 Signal 전송

```java
int tasks = 6;
Flux
    .create((FluxSink<String> sink) -> {
        IntStream
            .range(1, tasks)
            .forEach(n -> sink.next(doTask(n)));
    })
    .subscribeOn(Schedulers.boundedElastic())
    .doOnNext(n -> log.info("# create(): {}", n))
    .publishOn(Schedulers.parallel())
    .map(result -> result + " success!")
    .doOnNext(n -> log.info("# map(): {}", n))
    .publishOn(Schedulers.parallel())
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(500L);
```

#### Sinks를 사용하는 코드 예제

```java
int tasks = 6;

Sinks.Many<String> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
Flux<String> fluxView = unicastSink.asFlux();

IntStream
    .range(1, tasks)
    .forEach(n -> {
        try {
            new Thread(() -> {
                unicastSink.emitNext(doTask(n), FAIL_FAST);
                log.info("# emitted: {}", n);
            }).start();
            Thread.sleep(100L);
        } catch (InterruptedException e) {
            log.error(e.getMessage());
        }
    });

fluxView
    .publishOn(Schedulers.parallel())
    .map(result -> result + " success!")
    .doOnNext(n -> log.info("# map(): {}", n))
    .publishOn(Schedulers.parallel())
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(200L);
```

### 참고: Thread Safety

[What Is Thread-Safety and How to Achieve It? | Baeldung](https://www.baeldung.com/java-thread-safety)

---

## Sinks 종류 및 특징

### Sinks.One

- 한 건의 데이터를 전송하는 방법을 정의해 둔 기능 명세
- `Sinks.one()`
- `emitValue()`
  - 한 건의 데이터를 emit 하는 메서드
- `asMono()`
  - Sinks.One을 Mono로 변환하는 메서드

```java
Sinks.One<String> sinkOne = Sinks.one();
Mono<String> mono = sinkOne.asMono();

sinkOne.emitValue("Hello Reactor", FAIL_FAST);
sinkOne.emitValue("Hi Reactor", FAIL_FAST);
sinkOne.emitValue(null, FAIL_FAST);

mono.subscribe(data -> log.info("# Subscriber1 {}", data));
mono.subscribe(data -> log.info("# Subscriber2 {}", data));

/*
22:07:57.893 [main] DEBUG- onNextDropped: Hi Reactor
22:07:57.895 [main] INFO - # Subscriber1 Hello Reactor
22:07:57.896 [main] INFO - # Subscriber2 Hello Reactor
*/
```

---

### Sinks.Many

- 여러 건의 데이터를 여러 가지 방식으로 전송하는 기능을 정의해 둔 기능 명세
- `Sinks.many()`
  - 데이터 emit을 위한 여러 가지 기능이 정의된 ManySpec을 반환
- `asFlux()`
  - Sinks.Many를 Flux로 변환하는 메서드

#### UnicastSpec

- 단 하나의 Subscriber에게 데이터를 emit 한다.

```java
Sinks.Many<Integer> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
Flux<Integer> fluxView = unicastSink.asFlux();

unicastSink.emitNext(1, FAIL_FAST);
unicastSink.emitNext(2, FAIL_FAST);

fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

unicastSink.emitNext(3, FAIL_FAST);

/* 주석 제거 시, IllegalStateException 발생
Caused by: java.lang.IllegalStateException: UnicastProcessor allows only a single Subscriber */
// fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

/*
22:20:11.465 [main] INFO - # Subscriber1: 1
22:20:11.466 [main] INFO - # Subscriber1: 2
22:20:11.466 [main] INFO - # Subscriber1: 3
*/
```

#### MulticastSpec

- 하나 이상의 Subscriber에게 데이터를 emit한다.

```java
Sinks.Many<Integer> multicastSink =
        Sinks.many().multicast().onBackpressureBuffer();

Flux<Integer> fluxView = multicastSink.asFlux();

multicastSink.emitNext(1, FAIL_FAST);
multicastSink.emitNext(2, FAIL_FAST);

fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));
fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

multicastSink.emitNext(3, FAIL_FAST);

/*
22:34:19.836 [main] INFO - # Subscriber1: 1
22:34:19.837 [main] INFO - # Subscriber1: 2
22:34:19.838 [main] INFO - # Subscriber1: 3
22:34:19.838 [main] INFO - # Subscriber2: 3
*/
```

#### MulticastReplaySpec

- 하나 이상의 Subscriber에게 데이터를 emit한다.
- emit된 데이터 중에서 특정 시점으로 되돌린 데이터부터 emit한다. (`replay`)

```java
Sinks.Many<Integer> replaySink = Sinks.many().replay().limit(2);
Flux<Integer> fluxView = replaySink.asFlux();

replaySink.emitNext(1, FAIL_FAST);
replaySink.emitNext(2, FAIL_FAST);
replaySink.emitNext(3, FAIL_FAST);

fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

replaySink.emitNext(4, FAIL_FAST);

fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

/*
22:37:18.104 [main] INFO - # Subscriber1: 2
22:37:18.105 [main] INFO - # Subscriber1: 3
22:37:18.105 [main] INFO - # Subscriber1: 4
22:37:18.105 [main] INFO - # Subscriber2: 3
22:37:18.105 [main] INFO - # Subscriber2: 4
*/
```
