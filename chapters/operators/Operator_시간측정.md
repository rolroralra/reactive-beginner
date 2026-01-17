# Sequence의 동작 시간 측정을 위한 Operator
---
---
## elapsed
```java
public final Mono<Tuple2<Long,T>> elapsed()
public final Mono<Tuple2<Long,T>> elapsed(Scheduler scheduler)
public final Flux<Tuple2<Long,T>> elapsed()
public final Flux<Tuple2<Long,T>> elapsed(Scheduler scheduler)
```

![](../../images/elapsedForFlux.svg)

