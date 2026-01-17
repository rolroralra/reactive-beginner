# Sequence 필터링 Operator
---
---
## Flux.filter
```java
public final Flux<T> filter(Predicate<? super T> p)
```
- filter() Operator는 Upstream에서 emit된 데이터 중에서 조건에 일치하는 데이터만 Downstream으로 emit한다.
\![](../../images/filterForFlux.svg)
```java
Flux
      .range(1, 20)      .filter(num -> num % 2 != 0)      .subscribe(data -> log.info("# onNext: {}", data));/*18:15:59.367 [main] INFO - # onNext: 118:15:59.367 [main] INFO - # onNext: 318:15:59.367 [main] INFO - # onNext: 518:15:59.367 [main] INFO - # onNext: 718:15:59.368 [main] INFO - # onNext: 918:15:59.368 [main] INFO - # onNext: 1118:15:59.368 [main] INFO - # onNext: 1318:15:59.368 [main] INFO - # onNext: 1518:15:59.368 [main] INFO - # onNext: 1718:15:59.368 [main] INFO - # onNext: 19*/
```
## Flux.filterWhen
- filterWhen() Operator는 내부에서 Inner Sequence를 통해 조건에 맞는 데이터인지를 비동기적으로 필터링한다.
```java
public final Flux<T> filterWhen(Function<? super T,? extends Publisher<Boolean>> asyncPredicate)
```
\![](../../images/filterWhenForFlux.svg)
```java
Map<CovidVaccine, Tuple2<CovidVaccine, Integer>> vaccineMap = getCovidVaccines();Flux
    .fromIterable(SampleData.coronaVaccineNames)    .filterWhen(vaccine -> Mono
                            .just(vaccineMap.get(vaccine).getT2() >= 3_000_000)                            .publishOn(Schedulers.parallel()))    .subscribe(data -> log.info("# onNext: {}", data));Thread.sleep(1000);/*20:32:24.395 [parallel-2] INFO - # onNext: AstraZeneca20:32:24.399 [parallel-3] INFO - # onNext: Moderna*/
```
## Flux.skip
### Flux.skip(long skipped)
```java
public final Flux<T> skip(long skipped)
```
- skip() Operator는 Upstream에서 emit된 데이터 중에서 파라미터로 입력받은 숫자만큼 건너띈 후
\![](../../images/skip.svg)
```java
Flux
    .interval(Duration.ofSeconds(1))    .skip(2)    .subscribe(data -> log.info("# onNext: {}", data));Thread.sleep(5500L);/*20:35:25.068 [parallel-1] INFO - # onNext: 220:35:26.072 [parallel-1] INFO - # onNext: 320:35:27.072 [parallel-1] INFO - # onNext: 4*/
```
### Flux.skip(Duration timespan)
```java
public final Flux<T> skip(Duration timespan)public final Flux<T> skip(Duration timespan, Scheduler timer)
```
- skip() Operator의 파라미터로 시간을 지정하면,
\![](../../images/skipWithTimespan.svg)
```java
Flux
    .interval(Duration.ofMillis(300))    .skip(Duration.ofSeconds(1))    .subscribe(data -> log.info("# onNext: {}", data));Thread.sleep(2000L);
```
## Flux.take
### Flux.take(long n)
```java
public final Flux<T> take(long n)public final Flux<T> take(long n, boolean limitRequest)
```
- take() Operator는 Upstream에서 emit되는 데이터 중에서 파라미터로 입력받은 숫자만큼만 Downstream으로 emit한다.
\![](../../images/takeLimitRequestTrue.svg)
```java
Flux
    .interval(Duration.ofSeconds(1))    .take(3)    .subscribe(data -> log.info("# onNext: {}", data));Thread.sleep(4000L);/*20:48:23.138 [parallel-1] INFO - # onNext: 020:48:24.136 [parallel-1] INFO - # onNext: 120:48:25.134 [parallel-1] INFO - # onNext: 2*/
```
### Flux.take(Duration timespan)
```java
public final Flux<T> take(Duration timespan)public final Flux<T> take(Duration timespan, Scheduler timer)
```
- take() Operator의 파라미터로 시간을 지정하면,
\![](../../images/takeWithTimespanForFlux.svg)
```java
Flux
    .interval(Duration.ofSeconds(1))    .take(Duration.ofMillis(2500))    .subscribe(data -> log.info("# onNext: {}", data));Thread.sleep(3000L);/*20:49:35.549 [parallel-2] INFO - # onNext: 020:49:36.540 [parallel-2] INFO - # onNext: 1*/
```
### Flux.takeLast(int n)
```java
public final Flux<T> takeLast(int n)
```
- takeLast() Operator는 Upstream에서 emit된 데이터 중에서 파라미터로 입력한 개수만큼 가장 마지막에 emit된 데이터를 Downstream으로 emit한다.
\![](../../images/takeLast.svg)
```java
Flux
    .range(1, 10)    .takeLast(2)    .subscribe(data -> log.info("# data: {}", data));/*20:54:03.025 [main] INFO - # data: 920:54:03.026 [main] INFO - # data: 10*/
```
### Flux.takeUntil(Predicate<? super T> predicate)
```java
public final Flux<T> takeUntil(Predicate<? super T> predicate)public final Flux<T> takeUntilOther(Publisher<?> other)
```
- takeUntil() Operator는 파라미터로 입력한 Predicate가 true가 될 때까지
- ⚠️ Predicate를 평가할 때, 사용한 데이터가 포함된다.
\![](../../images/takeUntil.svg)
```java
Flux.range(1, 10)    .takeUntil(data -> data >= 5)    .subscribe(data -> log.info("# data: {}", data));/*20:57:08.898 [main] INFO - # data: 120:57:08.899 [main] INFO - # data: 220:57:08.899 [main] INFO - # data: 320:57:08.899 [main] INFO - # data: 420:57:08.899 [main] INFO - # data: 5*/
```
### Flux.takeWhile(Predicate<? super T> continuePredicate)
```java
public final Flux<T> takeWhile(Predicate<? super T> continuePredicate)
```
- takeWhile() Operator는 파라미터로 입력된 Predicate가 true인 동안에만
- ⚠️ Predicate를 평가시 false인 데이터는 Downstream으로 emit되지 않는다.
\![](../../images/takeWhile.svg)
```java
Flux.range(1, 10)    .takeWhile(data -> data < 5)    .subscribe(data -> log.info("# data: {}", data));/*21:01:19.118 [main] INFO - # data: 121:01:19.119 [main] INFO - # data: 221:01:19.119 [main] INFO - # data: 321:01:19.119 [main] INFO - # data: 4*/
```
## Flux.next
```java
public final Mono<T> next()
```
- next() Operator는 Upstream에서 emit되는 데이터 중에서 첫 번째 데이터만 Downstream으로 emit한다.
\![](../../images/next.svg)
```java
Flux.range(1, 10)    .next()    .subscribe(data -> log.info("# data: {}", data));
```

