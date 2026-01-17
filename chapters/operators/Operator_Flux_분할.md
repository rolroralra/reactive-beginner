# Flux Sequence 분할을 위한 Operator
---
---
## Flux.window
```java
public final Flux<Flux<T>> window(int maxSize)
public final Flux<Flux<T>> window(int maxSize, int skip)
public final Flux<Flux<T>> window(Publisher<?> boundary)
public final Flux<Flux<T>> window(Duration windowingTimespan)
public final Flux<Flux<T>> window(Duration windowingTimespan, Duration openWindowEvery)
public final Flux<Flux<T>> window(Duration windowingTimespan, Scheduler timer)
public final Flux<Flux<T>> window(Duration windowingTimespan, Duration openWindowEvery, Scheduler timer)
```

![](../../images/windowWithMaxSize.svg)

## Flux.buffer
```java
public final Flux<List<T>> buffer()
public final Flux<List<T>> buffer(int maxSize)
public final <C extends Collection<? super T>> Flux<C> buffer(int maxSize, Supplier<C> bufferSupplier)
public final Flux<List<T>> buffer(int maxSize, int skip)
public final <C extends Collection<? super T>> Flux<C> buffer(int maxSize, int skip, Supplier<C> bufferSupplier)
public final Flux<List<T>> buffer(Publisher<?> other)
public final <C extends Collection<? super T>> Flux<C> buffer(Publisher<?> other, Supplier<C> bufferSupplier)
public final Flux<List<T>> buffer(Duration bufferingTimespan)
public final Flux<List<T>> buffer(Duration bufferingTimespan, Duration openBufferEvery)
public final Flux<List<T>> buffer(Duration bufferingTimespan, Scheduler timer)
public final Flux<List<T>> buffer(Duration bufferingTimespan, Duration openBufferEvery, Scheduler timer)
```

![](../../images/bufferWithMaxSize.svg)

## Flux.bufferTimeout
```java
public final Flux<List<T>> bufferTimeout(int maxSize, Duration maxTime)
public final <C extends Collection<? super T>> Flux<C> bufferTimeout(int maxSize, Duration maxTime, Supplier<C> bufferSupplier)
public final Flux<List<T>> bufferTimeout(int maxSize, Duration maxTime, Scheduler timer)
public final <C extends Collection<? super T>> Flux<C> bufferTimeout(int maxSize, Duration maxTime, Scheduler timer, Supplier<C> bufferSupplier)
public final Flux<List<T>> bufferTimeout(int maxSize, Duration maxTime, boolean fairBackpressure)
public final Flux<List<T>> bufferTimeout(int maxSize, Duration maxTime, Scheduler timer, boolean fairBackpressure)
public final <C extends Collection<? super T>> Flux<C> bufferTimeout(int maxSize, Duration maxTime, Supplier<C> bufferSupplier, boolean fairBackpressure)
public final <C extends Collection<? super T>> Flux<C> bufferTimeout(int maxSize, Duration maxTime, Scheduler timer, Supplier<C> bufferSupplier, boolean fairBackpressure)
```

![](../../images/bufferTimeoutWithMaxSizeAndTimespan.svg)

## Flux.groupBy
```java
public final <K> Flux<GroupedFlux<K,T>> groupBy(Function<? super T,? extends K> keyMapper)
public final <K> Flux<GroupedFlux<K,T>> groupBy(Function<? super T,? extends K> keyMapper, int prefetch)
public final <K,V> Flux<GroupedFlux<K,V>> groupBy(Function<? super T,? extends K> keyMapper, Function<? super T,? extends V> valueMapper)
public final <K,V> Flux<GroupedFlux<K,V>> groupBy(Function<? super T,? extends K> keyMapper, Function<? super T,? extends V> valueMapper, int prefetch)
```

![](../../images/groupByWithKeyMapper.svg)

