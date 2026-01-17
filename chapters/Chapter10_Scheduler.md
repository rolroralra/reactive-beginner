# Chapter10. Scheduler

---

## Scheduler란?

> Reactor Sequence에서 사용되는 Thread를 관리해주는 관리자

---

## Scheduler를 위한 전용 Operator

### subscribeOn()

- 구독이 발생한 직후 실행될 스레드를 지정하는 Operator

```java
Flux.fromArray(new Integer[] {1, 3, 5, 7})
        .subscribeOn(Schedulers.boundedElastic())
        .doOnNext(data -> log.info("# doOnNext: {}", data))
        .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
        .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(500L);

/*
23:23:16.452 [main] INFO - # doOnSubscribe
23:23:16.455 [boundedElastic-1] INFO - # doOnNext: 1
23:23:16.456 [boundedElastic-1] INFO - # onNext: 1
23:23:16.456 [boundedElastic-1] INFO - # doOnNext: 3
23:23:16.456 [boundedElastic-1] INFO - # onNext: 3
23:23:16.456 [boundedElastic-1] INFO - # doOnNext: 5
23:23:16.456 [boundedElastic-1] INFO - # onNext: 5
23:23:16.456 [boundedElastic-1] INFO - # doOnNext: 7
23:23:16.456 [boundedElastic-1] INFO - # onNext: 7
*/
```

### publishOn()

- Downstream으로 Signal을 전송할 때 실행되는 스레드를 제어하는 역할을 하는 Operator
- publishOn()을 기준으로 Downstream의 실행 스레드를 변경한다.

```java
Flux.fromArray(new Integer[] {1, 3, 5, 7})
        .doOnNext(data -> log.info("# doOnNext: {}", data))
        .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
        .publishOn(Schedulers.parallel())
        .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(500L);

/*
23:31:21.160 [main] INFO - # doOnSubscribe
23:31:21.166 [main] INFO - # doOnNext: 1
23:31:21.168 [main] INFO - # doOnNext: 3
23:31:21.168 [parallel-1] INFO - # onNext: 1
23:31:21.168 [main] INFO - # doOnNext: 5
23:31:21.168 [main] INFO - # doOnNext: 7
23:31:21.172 [parallel-1] INFO - # onNext: 3
23:31:21.172 [parallel-1] INFO - # onNext: 5
23:31:21.173 [parallel-1] INFO - # onNext: 7
*/
```

### parallel()

- Round Robin 방식으로 CPU 코어 개수만큼의 스레드를 병렬로 실행한다.
- `parallel()`: Flux를 ParallelFlux로 변환
- `runOn()`: 실제 병렬 처리를 수행할 Scheduler를 지정

```java
Flux.fromArray(new Integer[]{1, 3, 5, 7, 9, 11, 13, 15, 17, 19})
        .parallel(4)
        .runOn(Schedulers.parallel())
        .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(100L);

/*
23:40:22.340 [parallel-1] INFO - # onNext: 1
23:40:22.340 [parallel-4] INFO - # onNext: 7
23:40:22.341 [parallel-3] INFO - # onNext: 5
23:40:22.340 [parallel-2] INFO - # onNext: 3
23:40:22.344 [parallel-3] INFO - # onNext: 13
23:40:22.345 [parallel-4] INFO - # onNext: 15
23:40:22.345 [parallel-1] INFO - # onNext: 9
23:40:22.345 [parallel-2] INFO - # onNext: 11
23:40:22.345 [parallel-1] INFO - # onNext: 17
23:40:22.345 [parallel-2] INFO - # onNext: 19
*/
```

---

## publishOn()과 subscribeOn()의 동작 이해

- `publishOn()` Operator는 한 개 이상 사용할 수 있다.
- `subscribeOn()` Operator와 `publishOn()` Operator를 함께 사용해서 원본 Publisher에서 데이터를 emit하는 스레드와 emit된 데이터를 가공 처리하는 스레드를 적절하게 분리할 수 있다.
- `subscribeOn()`는 Operator 체인상에서 어떤 위치에 있든 간에 구독 시점 직후에 실행될 스레드를 지정한다.

### subscribeOn()과 publishOn()을 사용하지 않은 경우

- Sequence의 Operator 체인에서 최초의 스레드는 subscribe()가 호출되는 scope에 있는 스레드

```java
Flux
    .fromArray(new Integer[] {1, 3, 5, 7})
    .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
    .filter(data -> data > 3)
    .doOnNext(data -> log.info("# doOnNext filter: {}", data))
    .map(data -> data * 10)
    .doOnNext(data -> log.info("# doOnNext map: {}", data))
    .subscribe(data -> log.info("# onNext: {}", data));

/*
19:33:03.233 [main] INFO - # doOnNext fromArray: 1
19:33:03.234 [main] INFO - # doOnNext fromArray: 3
19:33:03.234 [main] INFO - # doOnNext fromArray: 5
19:33:03.234 [main] INFO - # doOnNext filter: 5
19:33:03.234 [main] INFO - # doOnNext map: 50
19:33:03.235 [main] INFO - # onNext: 50
19:33:03.235 [main] INFO - # doOnNext fromArray: 7
19:33:03.235 [main] INFO - # doOnNext filter: 7
19:33:03.235 [main] INFO - # doOnNext map: 70
19:33:03.235 [main] INFO - # onNext: 70
*/
```

### 한 개의 publishOn()만 사용한 경우

- publishOn() 아래 쪽 Operator들의 실행 스레드를 변경한다.

```java
Flux
    .fromArray(new Integer[] {1, 3, 5, 7})
    .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
    .publishOn(Schedulers.parallel())
    .filter(data -> data > 3)
    .doOnNext(data -> log.info("# doOnNext filter: {}", data))
    .map(data -> data * 10)
    .doOnNext(data -> log.info("# doOnNext map: {}", data))
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(500L);

/*
19:36:14.638 [main] INFO - # doOnNext fromArray: 1
19:36:14.639 [main] INFO - # doOnNext fromArray: 3
19:36:14.639 [main] INFO - # doOnNext fromArray: 5
19:36:14.639 [main] INFO - # doOnNext fromArray: 7
19:36:14.639 [parallel-1] INFO - # doOnNext filter: 5
19:36:14.640 [parallel-1] INFO - # doOnNext map: 50
19:36:14.640 [parallel-1] INFO - # onNext: 50
19:36:14.640 [parallel-1] INFO - # doOnNext filter: 7
19:36:14.640 [parallel-1] INFO - # doOnNext map: 70
19:36:14.640 [parallel-1] INFO - # onNext: 70
*/
```

### 두 개의 publishOn()을 사용한 경우

- 다음 publishOn()을 만나기 전까지 publishOn() 아래 쪽 Operator들의 실행 스레드를 변경한다.

```java
Flux
    .fromArray(new Integer[] {1, 3, 5, 7})
    .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
    .publishOn(Schedulers.parallel())
    .filter(data -> data > 3)
    .doOnNext(data -> log.info("# doOnNext filter: {}", data))
    .publishOn(Schedulers.parallel())
    .map(data -> data * 10)
    .doOnNext(data -> log.info("# doOnNext map: {}", data))
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(500L);

/*
19:37:54.802 [main] INFO - # doOnNext fromArray: 1
19:37:54.803 [main] INFO - # doOnNext fromArray: 3
19:37:54.803 [main] INFO - # doOnNext fromArray: 5
19:37:54.803 [main] INFO - # doOnNext fromArray: 7
19:37:54.803 [parallel-2] INFO - # doOnNext filter: 5
19:37:54.804 [parallel-2] INFO - # doOnNext filter: 7
19:37:54.804 [parallel-1] INFO - # doOnNext map: 50
19:37:54.804 [parallel-1] INFO - # onNext: 50
19:37:54.804 [parallel-1] INFO - # doOnNext map: 70
19:37:54.804 [parallel-1] INFO - # onNext: 70
*/
```

### subscribeOn()과 publishOn()을 함께 사용한 경우

- subscribeOn()은 구독 직후에 실행될 스레드를 지정한다.

```java
Flux
    .fromArray(new Integer[] {1, 3, 5, 7})
    .subscribeOn(Schedulers.boundedElastic())
    .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
    .filter(data -> data > 3)
    .doOnNext(data -> log.info("# doOnNext filter: {}", data))
    .publishOn(Schedulers.parallel())
    .map(data -> data * 10)
    .doOnNext(data -> log.info("# doOnNext map: {}", data))
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(500L);

/*
19:40:02.915 [boundedElastic-1] INFO - # doOnNext fromArray: 1
19:40:02.916 [boundedElastic-1] INFO - # doOnNext fromArray: 3
19:40:02.917 [boundedElastic-1] INFO - # doOnNext fromArray: 5
19:40:02.917 [boundedElastic-1] INFO - # doOnNext filter: 5
19:40:02.917 [boundedElastic-1] INFO - # doOnNext fromArray: 7
19:40:02.917 [parallel-1] INFO - # doOnNext map: 50
19:40:02.917 [boundedElastic-1] INFO - # doOnNext filter: 7
19:40:02.917 [parallel-1] INFO - # onNext: 50
19:40:02.917 [parallel-1] INFO - # doOnNext map: 70
19:40:02.917 [parallel-1] INFO - # onNext: 70
*/
```

---

## Scheduler의 종류

### Schedulers.immediate()

- 별도의 스레드를 추가적으로 생성하지 않고, 현재 스레드에서 작업을 처리하고자 할 때 사용할 수 있다.

```java
Flux
    .fromArray(new Integer[] {1, 3, 5, 7})
    .publishOn(Schedulers.parallel())
    .filter(data -> data > 3)
    .doOnNext(data -> log.info("# doOnNext filter: {}", data))
    .publishOn(Schedulers.immediate())
    .map(data -> data * 10)
    .doOnNext(data -> log.info("# doOnNext map: {}", data))
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(200L);

/*
19:44:55.012 [parallel-1] INFO - # doOnNext filter: 5
19:44:55.013 [parallel-1] INFO - # doOnNext map: 50
19:44:55.014 [parallel-1] INFO - # onNext: 50
19:44:55.014 [parallel-1] INFO - # doOnNext filter: 7
19:44:55.014 [parallel-1] INFO - # doOnNext map: 70
19:44:55.014 [parallel-1] INFO - # onNext: 70
*/
```

### Schedulers.single()

- 스레드 하나만 생성해서, Scheduler가 제거되기 전까지 재사용하는 방식
- 하나의 스레드로 다수의 작업을 처리해야 되므로, 지연 시간이 짧은 작업을 처리하는 것이 효과적

```java
doTask("task1")
        .subscribe(data -> log.info("# onNext: {}", data));
doTask("task2")
        .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(200L);

private static Flux<Integer> doTask(String taskName) {
    return Flux.fromArray(new Integer[] {1, 3, 5, 7})
            .publishOn(Schedulers.single())
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# {} doOnNext filter: {}", taskName, data))
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# {} doOnNext map: {}", taskName, data));
}

/*
19:46:07.861 [single-1] INFO - # task1 doOnNext filter: 5
19:46:07.863 [single-1] INFO - # task1 doOnNext map: 50
19:46:07.863 [single-1] INFO - # onNext: 50
19:46:07.863 [single-1] INFO - # task1 doOnNext filter: 7
19:46:07.863 [single-1] INFO - # task1 doOnNext map: 70
19:46:07.863 [single-1] INFO - # onNext: 70
19:46:07.863 [single-1] INFO - # task2 doOnNext filter: 5
19:46:07.864 [single-1] INFO - # task2 doOnNext map: 50
19:46:07.864 [single-1] INFO - # onNext: 50
19:46:07.864 [single-1] INFO - # task2 doOnNext filter: 7
19:46:07.864 [single-1] INFO - # task2 doOnNext map: 70
19:46:07.864 [single-1] INFO - # onNext: 70
*/
```

### Schedulers.newSingle()

- 호출할 때마다 매번 새로운 스레드를 하나 생성한다.

```java
doTask("task1")
        .subscribe(data -> log.info("# onNext: {}", data));
doTask("task2")
        .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(200L);

private static Flux<Integer> doTask(String taskName) {
    return Flux.fromArray(new Integer[] {1, 3, 5, 7})
            .publishOn(Schedulers.newSingle("new-single", true))
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# {} doOnNext filter: {}", taskName, data))
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# {} doOnNext map: {}", taskName, data));
}

/*
19:46:50.784 [new-single-2] INFO - # task2 doOnNext filter: 5
19:46:50.784 [new-single-1] INFO - # task1 doOnNext filter: 5
19:46:50.786 [new-single-2] INFO - # task2 doOnNext map: 50
19:46:50.786 [new-single-1] INFO - # task1 doOnNext map: 50
19:46:50.786 [new-single-2] INFO - # onNext: 50
19:46:50.786 [new-single-1] INFO - # onNext: 50
19:46:50.786 [new-single-2] INFO - # task2 doOnNext filter: 7
19:46:50.786 [new-single-1] INFO - # task1 doOnNext filter: 7
19:46:50.786 [new-single-2] INFO - # task2 doOnNext map: 70
19:46:50.786 [new-single-1] INFO - # task1 doOnNext map: 70
19:46:50.786 [new-single-2] INFO - # onNext: 70
19:46:50.786 [new-single-1] INFO - # onNext: 70
*/
```

### Schedulers.boundedElastic()

- ExecutorService 기반의 Thread Pool을 생성한 후, 그 안에서 정해진 수만큼의 스레드를 사용하여 작업을 처리하고 작업이 종료된 스레드는 반납하여 재사용하는 방식
- **Blocking I/O 작업을 효과적으로 처리하기 위한 방식**
- 기본적으로 CPU 코어 수 x 10만큼의 스레드를 생성
- 100,000개의 작업이 큐에서 대기 가능

### Schedulers.parallel()

- **Non-Blocking I/O에 최적화되어 있는 Scheduler**로서 CPU 코어 수만큼의 스레드를 생성한다.

### Schedulers.fromExecutorService()

- 기존에 이미 사용하고 있는 ExecutorService가 있다면, 이 ExecutorService로부터 Scheduler를 생성하는 방식
- Reactor에서는 이 방식을 권장하지 않는다.

### Schedulers.newXXXX()

- Reactor에서 제공하는 디폴트 Scheduler 인스턴스를 사용하지 않고, 새로운 인스턴스를 생성할 수 있다.
- `Schedulers.newSingle()`, `Schedulers.newBoundedElastic()`, `Schedulers.newParallel()` 등
