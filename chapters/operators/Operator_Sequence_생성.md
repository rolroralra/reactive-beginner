# Sequence 생성을 위한 Operator
---
## Mono.justOrEmpty
```java
public static <T> Mono<T> justOrEmpty(@Nullable Optional<? extends T> data)

public static <T> Mono<T> justOrEmpty(@Nullable T data)
```
- just() 의 확장 Operator
- emit할 데이터가 null일 경우
\![](../../images/justOrEmpty.svg)
```java
Mono
    .justOrEmpty(null) // .justOrEmpty(Optional.empty())
    .subscribe(
        data -> {},
        error -> {},
        () -> log.info("# onComplete")
    );

/*
23:41:10.919 [main] INFO - # onComplete
*/
```
## Flux.fromIterable
```java
public static <T> Flux<T> fromIterable(Iterable<? extends T> it)
```
- Iterable 객체를 파라미터로 전달받아, 데이터를 순차적으로 emit하는 Flux를 생성
\![](../../images/fromIterable.svg)
```java
Flux
    .fromIterable(SampleData.coins)
    .subscribe(coin ->
        log.info("coin 명: {}, 현재가: {}", coin.getT1(), coin.getT2())
    );

/*
23:43:22.841 [main] INFO - coin 명: BTC, 현재가: 52000000
23:43:22.843 [main] INFO - coin 명: ETH, 현재가: 1720000
23:43:22.843 [main] INFO - coin 명: XRP, 현재가: 533
23:43:22.843 [main] INFO - coin 명: ICX, 현재가: 2080
23:43:22.843 [main] INFO - coin 명: EOS, 현재가: 4020
23:43:22.843 [main] INFO - coin 명: BCH, 현재가: 558000
*/
```
## Flux.fromStream
```java
public static <T> Flux<T> fromStream(Stream<? extends T> s)

public static <T> Flux<T> fromStream(Supplier<Stream<? extends T>> streamSupplier)
```
- Stream에 포함된 데이터를 emit하는 Flux를 생성
- Java의 Stream 특성상 Stream은 재사용할 수 없다.
\![](../../images/fromStream.svg)
```java
Flux
    .fromStream(SampleData.coinNames::stream)
    .filter(coin -> coin.equals("BTC") || coin.equals("ETH"))
    .subscribe(data -> log.info("{}", data));

/*
23:44:14.378 [main] INFO - BTC
23:44:14.379 [main] INFO - ETH
*/
```
## Flux.range
```java
public static Flux<Integer> range(int start, int count)
```
- n부터 1씩 증가한 연속된 수를 m개 emit하는 Flux를 생성한다.
\![](../../images/range.svg)
```java
Flux
    .range(5, 10)
    .subscribe(data -> log.info("{}", data));

/*
23:55:46.158 [main] INFO - 5
23:55:46.159 [main] INFO - 6
23:55:46.159 [main] INFO - 7
*/
```
## 🌟 Mono.defer, Flux.defer
```java
public static <T> Mono<T> defer(Supplier<? extends Mono<? extends T>> supplier)

public static <T> Flux<T> defer(Supplier<? extends Publisher<? extends T>> supplier)
```
- defer Operator를 선언한 시점에 데이터를 emit하는 것이 아니다.
- 구독하는 시점에 데이터를 emit하는 Flux 또는 Mono를 생성!
\![](https://prod-files-secure.s3.us-west-2.amazonaws.com/bb34ddaa-60bc-4801-b18f-492d9acd8316/62702c10-c89b-4635-aef8-22cfc06af82a/Untitled_2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466SBLA6HPT%2F20260117%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260117T052550Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEJT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIAPoJ8uLHhBe6P038wRZaURTWW2rre8d%2FLbw2GCXpZ69AiEApYxnz7wmaMSXe7dKmmwSI4lxFp%2Buh3sHP09GoJODTBoq%2FwMIXRAAGgw2Mzc0MjMxODM4MDUiDH%2FD1IjCYSk9MjCAJircAxtlb2duhZS89zW3iL0QQQPIdrlUY%2FEWcUzI2x%2BxWvKBG1eBC0480eUYiI1iXru8reLLGL4uPcRAc6Q8gs3rLGVaNzOJuSfl%2BoUAcp1%2FgUq1RMIeSSy9VfNEtn95DTGEXwkWrsT8rf0%2B46INHPZByozhowl6EefaHef8GVPAn1QqTl8MnQKX3qFfyEMK7KL%2BzkldeHoxF0J%2FHDQ3zJNJ2kNtSfWFF5w8jebs%2BdXGJmQzBIpqy9HJwbHZVInjEKba5pNJ23KsKvIH3e16L%2BMBwzd2tHXvn1kr4zPDwKYQ1HFdXZTgcrzoOqQ81FlSXk9WviqIXA20frXKj%2FJKK0wSgegfFcDAmFWdaWQ0ymdLheeh437lDdG3pvHysS326VvESg5PGPtc2%2BhE7URwYKwMRKHyoyI0WLiCHsg7tWa5nLM9y02N4doeCmjlDoHaXU1BfRl%2BxuqcEzNxFIyqxxVN25LMchkGM1vdvrZ3rXYcgEs1Db7wpTNoP4WiId5Z%2BBRAROXSeSmupArCi8ayb%2FmgsCZk2k0szx9uY2%2FJONyDGTVLAE3DbHWYbSA4FnZbQ7Sz33n4mqW4zU5iHe30ssH1UXh6Af73hwStuaHP514WsY7II1EJfJts2QNNmJKhMKSTrMsGOqUB6%2FS2iKp8Mn0tmE%2FkvpNym5zgxeSaNvSOmxskBGcNloc1uF1s%2FKEEyMBoSSgdckPTOVVrFAF5PMxffoghjFpcwDj7R8J3wta6JTVlA%2FifHIuZNjVhp1SvcLvcUcsE0QDuZLW6INb9CdNhW5cMcaV15ybkuiFwmGrg8hiC99X7ksFq%2FSdlkpfKfUK0eZHKXxNiwrUrQXkXrwItDHpu4bQYdCFL2je0&X-Amz-Signature=fdba01f65910e39312c46c93526bbd04b0c01aacfeac8375dcbe6ca5693577eb&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
Untitled
\![](https://prod-files-secure.s3.us-west-2.amazonaws.com/bb34ddaa-60bc-4801-b18f-492d9acd8316/29cf7dc0-4848-4fb8-ae92-5a9e66d3678f/Untitled_3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466SBLA6HPT%2F20260117%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260117T052550Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEJT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIAPoJ8uLHhBe6P038wRZaURTWW2rre8d%2FLbw2GCXpZ69AiEApYxnz7wmaMSXe7dKmmwSI4lxFp%2Buh3sHP09GoJODTBoq%2FwMIXRAAGgw2Mzc0MjMxODM4MDUiDH%2FD1IjCYSk9MjCAJircAxtlb2duhZS89zW3iL0QQQPIdrlUY%2FEWcUzI2x%2BxWvKBG1eBC0480eUYiI1iXru8reLLGL4uPcRAc6Q8gs3rLGVaNzOJuSfl%2BoUAcp1%2FgUq1RMIeSSy9VfNEtn95DTGEXwkWrsT8rf0%2B46INHPZByozhowl6EefaHef8GVPAn1QqTl8MnQKX3qFfyEMK7KL%2BzkldeHoxF0J%2FHDQ3zJNJ2kNtSfWFF5w8jebs%2BdXGJmQzBIpqy9HJwbHZVInjEKba5pNJ23KsKvIH3e16L%2BMBwzd2tHXvn1kr4zPDwKYQ1HFdXZTgcrzoOqQ81FlSXk9WviqIXA20frXKj%2FJKK0wSgegfFcDAmFWdaWQ0ymdLheeh437lDdG3pvHysS326VvESg5PGPtc2%2BhE7URwYKwMRKHyoyI0WLiCHsg7tWa5nLM9y02N4doeCmjlDoHaXU1BfRl%2BxuqcEzNxFIyqxxVN25LMchkGM1vdvrZ3rXYcgEs1Db7wpTNoP4WiId5Z%2BBRAROXSeSmupArCi8ayb%2FmgsCZk2k0szx9uY2%2FJONyDGTVLAE3DbHWYbSA4FnZbQ7Sz33n4mqW4zU5iHe30ssH1UXh6Af73hwStuaHP514WsY7II1EJfJts2QNNmJKhMKSTrMsGOqUB6%2FS2iKp8Mn0tmE%2FkvpNym5zgxeSaNvSOmxskBGcNloc1uF1s%2FKEEyMBoSSgdckPTOVVrFAF5PMxffoghjFpcwDj7R8J3wta6JTVlA%2FifHIuZNjVhp1SvcLvcUcsE0QDuZLW6INb9CdNhW5cMcaV15ybkuiFwmGrg8hiC99X7ksFq%2FSdlkpfKfUK0eZHKXxNiwrUrQXkXrwItDHpu4bQYdCFL2je0&X-Amz-Signature=f686b1061ddf3889d485ea78c29e644336948c87a1c08defb9188a0ecf53a0df&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
Untitled
```java
log.info("# start: {}", LocalDateTime.now());

Mono<LocalDateTime> justMono = Mono.just(LocalDateTime.now());
Mono<LocalDateTime> deferMono = Mono.defer(() -> Mono.just(LocalDateTime.now()));

Flux<LocalDateTime> deferFlux = Flux.defer(() ->
    Flux.just(
        LocalDateTime.now(),
        LocalDateTime.now(),
        LocalDateTime.now()
    )
);

Thread.sleep(2000);

justMono.subscribe(data -> log.info("# onNext just1: {}", data));
deferMono.subscribe(data -> log.info("# onNext defer1: {}", data));
deferFlux.subscribe(data -> log.info("# onNext deferFlux1: {}", data));

Thread.sleep(2000);

justMono.subscribe(data -> log.info("# onNext just2: {}", data));
deferMono.subscribe(data -> log.info("# onNext defer2: {}", data));
deferFlux.subscribe(data -> log.info("# onNext deferFlux2: {}", data));

/*
01:52:12.017 [main] INFO - # start: 2024-05-12T01:52:12.016151
01:52:14.102 [main] INFO - # onNext just1: 2024-05-12T01:52:12.019043
01:52:14.103 [main] INFO - # onNext defer1: 2024-05-12T01:52:14.103553
01:52:14.105 [main] INFO - # onNext deferFlux1: 2024-05-12T01:52:14.104601
01:52:14.105 [main] INFO - # onNext deferFlux1: 2024-05-12T01:52:14.104615
01:52:14.105 [main] INFO - # onNext deferFlux1: 2024-05-12T01:52:14.104619
01:52:16.109 [main] INFO - # onNext just2: 2024-05-12T01:52:12.019043
01:52:16.110 [main] INFO - # onNext defer2: 2024-05-12T01:52:16.110882
01:52:16.111 [main] INFO - # onNext deferFlux2: 2024-05-12T01:52:16.111611
01:52:16.111 [main] INFO - # onNext deferFlux2: 2024-05-12T01:52:16.111623
01:52:16.111 [main] INFO - # onNext deferFlux2: 2024-05-12T01:52:16.111627
*/
```
## 🌟 Flux.using
```java
public static <T, D> Flux<T> using(
    Callable<? extends D> resourceSupplier,
    Function<? super D, ? extends Publisher<? extends T>> sourceSupplier,
    Consumer<? super D> resourceCleanup
)
```
- 파라미터로 전달받은 resource를 emit하는 Flux를 생성
- 파라미터 목록
\![](https://prod-files-secure.s3.us-west-2.amazonaws.com/bb34ddaa-60bc-4801-b18f-492d9acd8316/d6e7e474-2310-46bd-8a66-440cda5842ba/Untitled_4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466SBLA6HPT%2F20260117%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260117T052550Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEJT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIAPoJ8uLHhBe6P038wRZaURTWW2rre8d%2FLbw2GCXpZ69AiEApYxnz7wmaMSXe7dKmmwSI4lxFp%2Buh3sHP09GoJODTBoq%2FwMIXRAAGgw2Mzc0MjMxODM4MDUiDH%2FD1IjCYSk9MjCAJircAxtlb2duhZS89zW3iL0QQQPIdrlUY%2FEWcUzI2x%2BxWvKBG1eBC0480eUYiI1iXru8reLLGL4uPcRAc6Q8gs3rLGVaNzOJuSfl%2BoUAcp1%2FgUq1RMIeSSy9VfNEtn95DTGEXwkWrsT8rf0%2B46INHPZByozhowl6EefaHef8GVPAn1QqTl8MnQKX3qFfyEMK7KL%2BzkldeHoxF0J%2FHDQ3zJNJ2kNtSfWFF5w8jebs%2BdXGJmQzBIpqy9HJwbHZVInjEKba5pNJ23KsKvIH3e16L%2BMBwzd2tHXvn1kr4zPDwKYQ1HFdXZTgcrzoOqQ81FlSXk9WviqIXA20frXKj%2FJKK0wSgegfFcDAmFWdaWQ0ymdLheeh437lDdG3pvHysS326VvESg5PGPtc2%2BhE7URwYKwMRKHyoyI0WLiCHsg7tWa5nLM9y02N4doeCmjlDoHaXU1BfRl%2BxuqcEzNxFIyqxxVN25LMchkGM1vdvrZ3rXYcgEs1Db7wpTNoP4WiId5Z%2BBRAROXSeSmupArCi8ayb%2FmgsCZk2k0szx9uY2%2FJONyDGTVLAE3DbHWYbSA4FnZbQ7Sz33n4mqW4zU5iHe30ssH1UXh6Af73hwStuaHP514WsY7II1EJfJts2QNNmJKhMKSTrMsGOqUB6%2FS2iKp8Mn0tmE%2FkvpNym5zgxeSaNvSOmxskBGcNloc1uF1s%2FKEEyMBoSSgdckPTOVVrFAF5PMxffoghjFpcwDj7R8J3wta6JTVlA%2FifHIuZNjVhp1SvcLvcUcsE0QDuZLW6INb9CdNhW5cMcaV15ybkuiFwmGrg8hiC99X7ksFq%2FSdlkpfKfUK0eZHKXxNiwrUrQXkXrwItDHpu4bQYdCFL2je0&X-Amz-Signature=8be2997a288ed92f39e8c2c96466f4929a2c5a67a3f3823e36195037c2afe472&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
Untitled
```java
URL resource = Example14_5.class.getResource("using_example.txt");
assert resource != null;

Path path = Paths.get(resource.toURI());

Flux.using(() -> Files.lines(path), Flux::fromStream, Stream::close)
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe(log::info);

Thread.sleep(5000);

/*
02:23:00.739 [boundedElastic-1] INFO - Hello, world!
02:23:00.740 [boundedElastic-1] INFO - Nice to meet you!
02:23:00.740 [boundedElastic-1] INFO - Good bye~
*/
```
## 🌟 Flux.generate
```java
public static <T, S> Flux<T> generate(
    Callable<S> stateSupplier,
    BiFunction<S, SynchronousSink<T>, S> generator
)
```
- 프로그래밍 방식으로 Signal 이벤트를 발생
- 특히 동기적으로 데이터를 하나씩 순차적으로 emit하고자 할 경우 사용
- SynchronousSink 는 하나의 Signal만 동기적으로 발생시킬수 있다.
\![](https://prod-files-secure.s3.us-west-2.amazonaws.com/bb34ddaa-60bc-4801-b18f-492d9acd8316/4526f2fc-afed-472a-8f4f-fa3b96ae6cd6/Untitled_5.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466SBLA6HPT%2F20260117%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260117T052550Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEJT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIAPoJ8uLHhBe6P038wRZaURTWW2rre8d%2FLbw2GCXpZ69AiEApYxnz7wmaMSXe7dKmmwSI4lxFp%2Buh3sHP09GoJODTBoq%2FwMIXRAAGgw2Mzc0MjMxODM4MDUiDH%2FD1IjCYSk9MjCAJircAxtlb2duhZS89zW3iL0QQQPIdrlUY%2FEWcUzI2x%2BxWvKBG1eBC0480eUYiI1iXru8reLLGL4uPcRAc6Q8gs3rLGVaNzOJuSfl%2BoUAcp1%2FgUq1RMIeSSy9VfNEtn95DTGEXwkWrsT8rf0%2B46INHPZByozhowl6EefaHef8GVPAn1QqTl8MnQKX3qFfyEMK7KL%2BzkldeHoxF0J%2FHDQ3zJNJ2kNtSfWFF5w8jebs%2BdXGJmQzBIpqy9HJwbHZVInjEKba5pNJ23KsKvIH3e16L%2BMBwzd2tHXvn1kr4zPDwKYQ1HFdXZTgcrzoOqQ81FlSXk9WviqIXA20frXKj%2FJKK0wSgegfFcDAmFWdaWQ0ymdLheeh437lDdG3pvHysS326VvESg5PGPtc2%2BhE7URwYKwMRKHyoyI0WLiCHsg7tWa5nLM9y02N4doeCmjlDoHaXU1BfRl%2BxuqcEzNxFIyqxxVN25LMchkGM1vdvrZ3rXYcgEs1Db7wpTNoP4WiId5Z%2BBRAROXSeSmupArCi8ayb%2FmgsCZk2k0szx9uY2%2FJONyDGTVLAE3DbHWYbSA4FnZbQ7Sz33n4mqW4zU5iHe30ssH1UXh6Af73hwStuaHP514WsY7II1EJfJts2QNNmJKhMKSTrMsGOqUB6%2FS2iKp8Mn0tmE%2FkvpNym5zgxeSaNvSOmxskBGcNloc1uF1s%2FKEEyMBoSSgdckPTOVVrFAF5PMxffoghjFpcwDj7R8J3wta6JTVlA%2FifHIuZNjVhp1SvcLvcUcsE0QDuZLW6INb9CdNhW5cMcaV15ybkuiFwmGrg8hiC99X7ksFq%2FSdlkpfKfUK0eZHKXxNiwrUrQXkXrwItDHpu4bQYdCFL2je0&X-Amz-Signature=d38f09479255289d12a2df99cc17cdaa6b684314170335d910d9e9ced158621e&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
Untitled
```java
Flux.generate(() -> 0, (state, sink) -> {
        sink.next(state);

        if (state == 10) {
            sink.complete();
        }

        return ++state;
    })
    .subscribe(data -> log.info("# onNext: {}", data));

/*
02:22:18.608 [main] INFO - # onNext: 0
02:22:18.608 [main] INFO - # onNext: 1
02:22:18.608 [main] INFO - # onNext: 2
02:22:18.609 [main] INFO - # onNext: 3
02:22:18.609 [main] INFO - # onNext: 4
02:22:18.609 [main] INFO - # onNext: 5
02:22:18.609 [main] INFO - # onNext: 6
02:22:18.609 [main] INFO - # onNext: 7
02:22:18.609 [main] INFO - # onNext: 8
02:22:18.609 [main] INFO - # onNext: 9
02:22:18.609 [main] INFO - # onNext: 10
*/
```
```java
final int dan = 3;

Flux.generate(
        () -> Tuples.of(dan, 1),
        (state, sink) -> {
            sink.next(
                state.getT1() + " * " + state.getT2() +
                    " = " + state.getT1() * state.getT2()
            );

            if (state.getT2() == 9) {
                sink.complete();
            }

            return Tuples.of(state.getT1(), state.getT2() + 1);
        },
        state -> log.info("# 구구단 {}단 종료!", state.getT1())
    )
    .subscribe(data -> log.info("# onNext: {}", data));

/*
02:34:46.242 [main] INFO - # onNext: 3 * 1 = 3
02:34:46.243 [main] INFO - # onNext: 3 * 2 = 6
02:34:46.243 [main] INFO - # onNext: 3 * 3 = 9
02:34:46.244 [main] INFO - # onNext: 3 * 4 = 12
02:34:46.244 [main] INFO - # onNext: 3 * 5 = 15
02:34:46.244 [main] INFO - # onNext: 3 * 6 = 18
02:34:46.244 [main] INFO - # onNext: 3 * 7 = 21
02:34:46.244 [main] INFO - # onNext: 3 * 8 = 24
02:34:46.244 [main] INFO - # onNext: 3 * 9 = 27
02:34:46.244 [main] INFO - # 구구단 3단 종료!
*/
```
## 🌟 Flux.create
```java
public static <T> Flux<T> create(Consumer<? super FluxSink<T>> emitter)

public static <T> Flux<T> create(
    Consumer<? super FluxSink<T>> emitter,
    FluxSink.OverflowStrategy backpressure
)
```
- create() Operator는 generate() Operator처럼 프로그래밍 방식으로 Signal 이벤트를 발생
- https://projectreactor.io/docs/core/release/api/reactor/core/publisher/FluxSink.html
### Flux.generate와 차이점
- generate의 경우, 데이터를 동기적으로 한 번에 한 건씩 emit할 수 있다.
- create의 경우, 한 번에 여러 건의 데이터를 비동기적으로 emit할 수 있다.
\![](https://prod-files-secure.s3.us-west-2.amazonaws.com/bb34ddaa-60bc-4801-b18f-492d9acd8316/9cd806b2-e1cb-4c91-865a-e3a414029053/Untitled_6.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466SBLA6HPT%2F20260117%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260117T052550Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEJT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIAPoJ8uLHhBe6P038wRZaURTWW2rre8d%2FLbw2GCXpZ69AiEApYxnz7wmaMSXe7dKmmwSI4lxFp%2Buh3sHP09GoJODTBoq%2FwMIXRAAGgw2Mzc0MjMxODM4MDUiDH%2FD1IjCYSk9MjCAJircAxtlb2duhZS89zW3iL0QQQPIdrlUY%2FEWcUzI2x%2BxWvKBG1eBC0480eUYiI1iXru8reLLGL4uPcRAc6Q8gs3rLGVaNzOJuSfl%2BoUAcp1%2FgUq1RMIeSSy9VfNEtn95DTGEXwkWrsT8rf0%2B46INHPZByozhowl6EefaHef8GVPAn1QqTl8MnQKX3qFfyEMK7KL%2BzkldeHoxF0J%2FHDQ3zJNJ2kNtSfWFF5w8jebs%2BdXGJmQzBIpqy9HJwbHZVInjEKba5pNJ23KsKvIH3e16L%2BMBwzd2tHXvn1kr4zPDwKYQ1HFdXZTgcrzoOqQ81FlSXk9WviqIXA20frXKj%2FJKK0wSgegfFcDAmFWdaWQ0ymdLheeh437lDdG3pvHysS326VvESg5PGPtc2%2BhE7URwYKwMRKHyoyI0WLiCHsg7tWa5nLM9y02N4doeCmjlDoHaXU1BfRl%2BxuqcEzNxFIyqxxVN25LMchkGM1vdvrZ3rXYcgEs1Db7wpTNoP4WiId5Z%2BBRAROXSeSmupArCi8ayb%2FmgsCZk2k0szx9uY2%2FJONyDGTVLAE3DbHWYbSA4FnZbQ7Sz33n4mqW4zU5iHe30ssH1UXh6Af73hwStuaHP514WsY7II1EJfJts2QNNmJKhMKSTrMsGOqUB6%2FS2iKp8Mn0tmE%2FkvpNym5zgxeSaNvSOmxskBGcNloc1uF1s%2FKEEyMBoSSgdckPTOVVrFAF5PMxffoghjFpcwDj7R8J3wta6JTVlA%2FifHIuZNjVhp1SvcLvcUcsE0QDuZLW6INb9CdNhW5cMcaV15ybkuiFwmGrg8hiC99X7ksFq%2FSdlkpfKfUK0eZHKXxNiwrUrQXkXrwItDHpu4bQYdCFL2je0&X-Amz-Signature=da17f9752b0e0c601387fceee0008c5a82c76ef1ae6cd09a74439d19ae9db5bb&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
Untitled
- 코드 예시 1 (request() 메서드를 활용한 pull 방식)
- 코드 예시 2 (Subscriber의 요청과 상관없이 비동기적으로 데이터를 emit하는 push 방식)
- 코드 예시 3 (Backpressure 전략 설정)
## 요약 정리
- just() Operator는 Hot Publisher 이기 때문에 Subscriber의 구독 여부와는 상관없이 데이터를 emit하며, 구독이 발생하면 emit된 데이터를 다시 재생(replay)해서 Subscriber에게 전달한다.
- defer() Operator는 구독이 발생하기 전까지 데이터의 emit을 지연시킨다.
- using() Operator는 파라미터로 전달받은 resource를 emit하는 Flux를 생성한다.
- generate() Operator는 프로그래밍 방식으로 Signal 이벤트를 발생시키며, 동기적으로 데이터를 하나씩 순차적으로 emit 한다.
- create() Operator 는 genreate() Operator와 마같가지로 프로그래밍 방식으로 Signal 이벤트를 발생시킨다.

