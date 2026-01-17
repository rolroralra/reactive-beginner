# Sequence 내부 동작 확인을 위한 Operator
---
---
## doOnXXXX()

### 코드예시
```java
Flux.range(1, 5)
    .doFinally(signalType -> log.info("# doFinally 1: {}", signalType))
    .doFinally(signalType -> log.info("# doFinally 2: {}", signalType))
    .doOnNext(data -> log.info("# range > doOnNext(): {}", data))
    .doOnRequest(data -> log.info("# doOnRequest: {}", data))
    .doOnSubscribe(subscription -> log.info("# doOnSubscribe 1"))
    .doFirst(() -> log.info("# doFirst()"))
    .filter(num -> num % 2 == 1)
    .doOnNext(data -> log.info("# filter > doOnNext(): {}", data))
    .doOnComplete(() -> log.info("# doOnComplete()"))
    .subscribe(new BaseSubscriber<>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(1);
        }
        @Override
        protected void hookOnNext(Integer value) {
            log.info("# hookOnNext: {}", value);
            request(1);
        }
    });

/*
21:58:27.135 [main] INFO - # doFirst()
21:58:27.138 [main] INFO - # doOnSubscribe 1
21:58:27.138 [main] INFO - # doOnRequest: 1
21:58:27.139 [main] INFO - # range > doOnNext(): 1
21:58:27.139 [main] INFO - # filter > doOnNext(): 1
21:58:27.139 [main] INFO - # hookOnNext: 1
21:58:27.139 [main] INFO - # doOnRequest: 1
21:58:27.139 [main] INFO - # range > doOnNext(): 2
21:58:27.139 [main] INFO - # range > doOnNext(): 3
21:58:27.139 [main] INFO - # filter > doOnNext(): 3
21:58:27.140 [main] INFO - # hookOnNext: 3
21:58:27.140 [main] INFO - # doOnRequest: 1
21:58:27.140 [main] INFO - # range > doOnNext(): 4
21:58:27.140 [main] INFO - # range > doOnNext(): 5
21:58:27.140 [main] INFO - # filter > doOnNext(): 5
21:58:27.140 [main] INFO - # hookOnNext: 5
21:58:27.140 [main] INFO - # doOnRequest: 1
21:58:27.140 [main] INFO - # doOnComplete()
21:58:27.140 [main] INFO - # doFinally 2: onComplete
21:58:27.140 [main] INFO - # doFinally 1: onComplete
*/
```

## doOnSubscribe
```java
public final Mono<T> doOnSubscribe(Consumer<? super Subscription> onSubscribe)
public final Flux<T> doOnSubscribe(Consumer<? super Subscription> onSubscribe)
```

![](../../images/doOnSubscribe.svg)

## doOnRequest
```java
public final Mono<T> doOnRequest(LongConsumer consumer)
public final Flux<T> doOnRequest(LongConsumer consumer)
```

![](../../images/doOnRequestForFlux.svg)

## doOnNext
```java
public final Mono<T> doOnNext(Consumer<? super T> onNext)
public final Flux<T> doOnNext(Consumer<? super T> onNext)
```

![](../../images/doOnNextForFlux.svg)

## doOnComplete
```java
public final Flux<T> doOnComplete(Runnable onComplete)
```

![](../../images/doOnComplete.svg)

## doOnError
```java
public final Mono<T> doOnError(Consumer<? super Throwable> onError)
public final <E extends Throwable> Mono<T> doOnError(Class<E> exceptionType, Consumer<? super E> onError)
public final Mono<T> doOnError(Predicate<? super Throwable> predicate, Consumer<? super Throwable> onError)
public final Flux<T> doOnError(Consumer<? super Throwable> onError)
public final <E extends Throwable> Flux<T> doOnError(Class<E> exceptionType, Consumer<? super E> onError)
public final Flux<T> doOnError(Predicate<? super Throwable> predicate, Consumer<? super Throwable> onError)
```

![](../../images/doOnErrorForFlux.svg)

## doOnCancel
```java
public final Mono<T> doOnCancel(Runnable onCancel)
public final Flux<T> doOnCancel(Runnable onCancel)
```

![](../../images/doOnCancelForFlux.svg)

## doOnTerminate
```java
public final Mono<T> doOnTerminate(Runnable onTerminate)
public final Flux<T> doOnTerminate(Runnable onTerminate)
```

![](../../images/doOnTerminateForFlux.svg)

## doOnEach
```java
public final Mono<T> doOnEach(Consumer<? super Signal<T>> signalConsumer)
public final Flux<T> doOnEach(Consumer<? super Signal<T>> signalConsumer)
```

![](../../images/doOnEachForFlux.svg)

## doOnDiscard
```java
public final <R> Mono<T> doOnDiscard(Class<R> type, Consumer<? super R> discardHook)
public final <R> Flux<T> doOnDiscard(Class<R> type, Consumer<? super R> discardHook)
```

## doAfterTerminate
```java
public final Mono<T> doAfterTerminate(Runnable afterTerminate)
public final Flux<T> doAfterTerminate(Runnable afterTerminate)
```

![](../../images/doAfterTerminateForFlux.svg)

## doFirst
```java
public final Mono<T> doFirst(Runnable onFirst)
public final Flux<T> doFirst(Runnable onFirst)
```

![](../../images/doFirstForFlux.svg)

```java
Flux.just(1, 2)
     .doFirst(() -> System.out.println("three"))
     .doFirst(() -> System.out.println("two"))
     .doFirst(() -> System.out.println("one")); //would print one two three
```

## doFinally
```java
public final Mono<T> doFinally(Consumer<SignalType> onFinally)
public final Flux<T> doFinally(Consumer<SignalType> onFinally)
```

![](../../images/doFinallyForMono.svg)

