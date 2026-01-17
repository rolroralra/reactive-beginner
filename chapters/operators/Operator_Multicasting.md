# 다수의 Subscriber에게 Flux를 Multicasting하기 위한 Operator
---
---
## publish
```java
public final ConnectableFlux<T> publish()
public final ConnectableFlux<T> publish(int prefetch)
public final <R> Flux<R> publish(Function<? super Flux<T>,? extends Publisher<? extends R>> transform)
public final <R> Flux<R> publish(Function<? super Flux<T>,? extends Publisher<? extends R>> transform, int prefetch)
```

![](../../images/publish.svg)

## ConnectableFlux.autoConnect
```java
public final Flux<T> autoConnect()
public final Flux<T> autoConnect(int minSubscribers)
public final Flux<T> autoConnect(int minSubscribers,
                                 Consumer<? super Disposable> cancelSupport)
```

![](../../images/autoConnectWithMinSubscribers.svg)

## ConnectableFlux.refCount
```java
public final Flux<T> refCount()
public final Flux<T> refCount(int minSubscribers)
public final Flux<T> refCount(int minSubscribers, Duration gracePeriod)
public final Flux<T> refCount(int minSubscribers, Duration gracePeriod, Scheduler scheduler)
```

![](../../images/refCountWithMinSubscribers.svg)

