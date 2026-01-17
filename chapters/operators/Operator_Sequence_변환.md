# Sequence 변환 Operator
---
---
## Flux.map
```java
public final <V> Flux<V> map(Function<? super T,? extends V> mapper)
public final <V> Flux<V> mapNotNull(Function<? super T,? extends V> mapper)
```
- `map()` Operator는 Upstream에서 emit된 데이터를 **mapper Function을 사용하여 변환**한 후, Downstream으로 emit 한다.

![](../../images/mapForFlux.svg)

```java
Flux
    .just("1-Circle", "3-Circle", "5-Circle")
    .map(circle -> circle.replace("Circle", "Rectangle"))
    .subscribe(data -> log.info("# onNext: {}", data));

/*
22:54:37.601 [main] INFO - # onNext: 1-Rectangle
22:54:37.602 [main] INFO - # onNext: 3-Rectangle
22:54:37.602 [main] INFO - # onNext: 5-Rectangle
*/
```

## Flux.flatMap
```java
public final <R> Flux<R> flatMap(Function<? super T,? extends Publisher<? extends R>> mapper)
public final <V> Flux<V> flatMap(Function<? super T,? extends Publisher<? extends V>> mapper, int concurrency)
public final <V> Flux<V> flatMap(Function<? super T,? extends Publisher<? extends V>> mapper, int concurrency, int prefetch)
```

![](../../images/flatMapForFlux.svg)

### 코드 예시 1
```java
Flux
    .just("Good", "Bad")
    .flatMap(feeling -> Flux
                            .just("Morning", "Afternoon", "Evening")
                            .map(time -> feeling + " " + time))
    .subscribe(log::info);

/*
23:09:19.498 [main] INFO - Good Morning
23:09:19.498 [main] INFO - Good Afternoon
23:09:19.498 [main] INFO - Good Evening
23:09:19.498 [main] INFO - Bad Morning
23:09:19.499 [main] INFO - Bad Afternoon
23:09:19.499 [main] INFO - Bad Evening
*/
```

### 코드 예시2
- `flatMap()` 내부의 Inner Sequence에 Scheduler를 설정해서 **비동기적으로** 데이터를 emit한 경우

```java
Flux
    .range(2, 8)
    .flatMap(dan -> Flux
                        .range(1, 9)
                        .publishOn(Schedulers.parallel())
                        .map(n -> dan + " * " + n + " = " + dan * n))
    .subscribe(log::info);

Thread.sleep(100L);

/*
23:12:12.066 [parallel-4] INFO - 5 * 1 = 5
23:12:12.068 [parallel-4] INFO - 5 * 2 = 10
23:12:12.068 [parallel-4] INFO - 5 * 3 = 15
23:12:12.068 [parallel-4] INFO - 5 * 4 = 20
23:12:12.068 [parallel-4] INFO - 3 * 1 = 3
23:12:12.068 [parallel-4] INFO - 3 * 2 = 6
...
*/
```

## Flux.concat

## Flux.merge

## Flux.zip

## Mono.and

## Flux.collectList

## Flux.collectMap

