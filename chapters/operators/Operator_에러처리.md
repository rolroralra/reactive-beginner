# 에러 처리를 위한 Operator
---
---
## error
```java
public static <T> Mono<T> error(Throwable error)
public static <T> Mono<T> error(Supplier<? extends Throwable> errorSupplier)
public static <T> Flux<T> error(Throwable error)
public static <T> Flux<T> error(Supplier<? extends Throwable> errorSupplier)
public static <O> Flux<O> error(Throwable throwable, boolean whenRequested)
```

![](../../images/error.svg)

## onErrorReturn
```java
public final Mono<T> onErrorReturn(T fallbackValue)
public final <E extends Throwable> Mono<T> onErrorReturn(Class<E> type, T fallbackValue)
public final Mono<T> onErrorReturn(Predicate<? super Throwable> predicate, T fallbackValue)
public final Flux<T> onErrorReturn(T fallbackValue)
public final <E extends Throwable> Flux<T> onErrorReturn(Class<E> type, T fallbackValue)
public final Flux<T> onErrorReturn(Predicate<? super Throwable> predicate, T fallbackValue)
```

![](../../images/onErrorReturnForFlux.svg)

## onErrorResume
```java
public final <E extends Throwable> Mono<T> onErrorResume(Class<E> type, Function<? super E,? extends Mono<? extends T>> fallback)
public final Mono<T> onErrorResume(Predicate<? super Throwable> predicate, Function<? super Throwable,? extends Mono<? extends T>> fallback)
public final Flux<T> onErrorResume(Function<? super Throwable,? extends Publisher<? extends T>> fallback)
public final <E extends Throwable> Flux<T> onErrorResume(Class<E> type, Function<? super E,? extends Publisher<? extends T>> fallback)
```

![](../../images/onErrorResumeForFlux.svg)

## onErrorContinue
```java
public final Mono<T> onErrorContinue(BiConsumer<Throwable,Object> errorConsumer)
public final <E extends Throwable> Mono<T> onErrorContinue(Class<E> type, BiConsumer<Throwable,Object> errorConsumer)
public final <E extends Throwable> Mono<T> onErrorContinue(Predicate<E> errorPredicate, BiConsumer<Throwable,Object> errorConsumer)
public final Flux<T> onErrorContinue(BiConsumer<Throwable,Object> errorConsumer)
public final <E extends Throwable> Flux<T> onErrorContinue(Class<E> type, BiConsumer<Throwable,Object> errorConsumer)
public final <E extends Throwable> Flux<T> onErrorContinue(Predicate<E> errorPredicate, BiConsumer<Throwable,Object> errorConsumer)
```

![](../../images/onErrorContinue.svg)

## retry
```java
public final Mono<T> retry()
public final Mono<T> retry(long numRetries)
public final Flux<T> retry()
public final Flux<T> retry(long numRetries)
```

![](../../images/retryWithAttemptsForFlux.svg)

