# Chapter08. Backpressure

---

## Backpressure란?

> Publisher가 끊임없이 emit하는 무수히 많은 데이터를 적절하게 제어하여 데이터 처리에 과부하가 걸리지 않도록 제어하는 데이터 처리 방식

---

## Reactor에서의 Backpressure 처리 방식

### 1. 데이터 개수 제어

> Subscriber가 적절히 처리할 수 있는 수준의 데이터 개수를 Publisher에게 요청하는 방식

```java
Flux.range(1, 5)
    .doOnRequest(data -> log.info("# doOnRequest: {}", data))
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(1);
        }

        @SneakyThrows
        @Override
        protected void hookOnNext(Integer value) {
            Thread.sleep(2000L);
            log.info("# hookOnNext: {}", value);
            request(1);
        }
    });
}

// 02:12:48.282 [main] INFO - # doOnRequest: 1
// 02:12:50.287 [main] INFO - # hookOnNext: 1
// 02:12:50.288 [main] INFO - # doOnRequest: 1
// 02:12:52.293 [main] INFO - # hookOnNext: 2
// 02:12:52.294 [main] INFO - # doOnRequest: 1
// 02:12:54.301 [main] INFO - # hookOnNext: 3
// 02:12:54.303 [main] INFO - # doOnRequest: 1
// 02:12:56.309 [main] INFO - # hookOnNext: 4
// 02:12:56.311 [main] INFO - # doOnRequest: 1
// 02:12:58.312 [main] INFO - # hookOnNext: 5
// 02:12:58.313 [main] INFO - # doOnRequest: 1
```

### 2. Backpressure 전략 사용

| 전략 | 설명 |
|------|------|
| IGNORE | Backpressure를 적용하지 않는 전략 |
| ERROR | Downstream의 데이터 처리 속도가 느릴 경우 Exception 발생 |
| DROP | Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우, 버퍼 밖에서 대기하는 먼저 emit된 데이터부터 Drop |
| LATEST | Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우, 버퍼 밖에서 대기하는 가장 최근에 emit된 데이터부터 버퍼에 채움 |
| BUFFER | Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우, 버퍼 안에 있는 데이터부터 Drop |

---

## IGNORE 전략

- Backpressure를 적용하지 않는 전략

---

## ERROR 전략

- Downstream의 데이터 처리 속도가 느려서, Upstream의 emit 속도를 따라가지 못할 경우
  - IllegalStateException 발생

```java
Flux
    .interval(Duration.ofMillis(1L))
    .onBackpressureError()
    .doOnNext(data -> log.info("# doOnNext: {}", data))
    .publishOn(Schedulers.parallel())
    .subscribe(data -> {
            try {
                Thread.sleep(5L);
            } catch (InterruptedException e) {}
            log.info("# onNext: {}", data);
        },
        error -> log.error("# onError", error));

Thread.sleep(2000L);

/*
13:38:32.009 [parallel-2] INFO - # doOnNext: 0
13:38:32.011 [parallel-2] INFO - # doOnNext: 1
13:38:32.011 [parallel-2] INFO - # doOnNext: 2
...
13:38:32.017 [parallel-1] INFO - # onNext: 0
...
13:38:33.683 [parallel-1] INFO - # onNext: 255
13:38:33.687 [parallel-1] ERROR- # onError
reactor.core.Exceptions$OverflowException: The receiver is overrun by more signals than expected (bounded queue...)
    at reactor.core.Exceptions.failWithOverflow(Exceptions.java:220)
    ...
*/
```

- `interval()` Operator
  - 지정한 시간 간격으로 0부터 1씩 증가한 숫자를 emit
- `onBackpressureError()`
  - ERROR 전략을 사용하는 Operator
- `doOnNext()` Operator
  - Publisher가 emit한 데이터를 확인하거나 추가적인 동작을 정의하는 용도로 사용
- `publishOn()` Operator
  - 별도의 스레드가 하나 더 실행된다.
- OverflowException
  - Downstream으로 전달할 데이터가 버퍼에 가득 찬 상태에서 추가적인 데이터가 emit 되면 발생하는 예외

---

## DROP 전략

- Publisher가 Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우
  - 버퍼 밖에서 대기 중인 데이터 중에서 먼저 emit 된 데이터부터 Drop 시킨다.
- **버퍼가 가득 찬 상태에서는 버퍼가 비워질 때까지 데이터를 Drop 한다.**

```java
Flux
    .interval(Duration.ofMillis(1L))
    .onBackpressureDrop(dropped -> log.info("# dropped: {}", dropped))
    .publishOn(Schedulers.parallel())
    .subscribe(data -> {
            try {
                Thread.sleep(5L);
            } catch (InterruptedException e) {}
            log.info("# onNext: {}", data);
        },
        error -> log.error("# onError", error));

Thread.sleep(2000L);

/*
14:31:14.551 [parallel-1] INFO - # onNext: 0
14:31:14.558 [parallel-1] INFO - # onNext: 1
14:31:14.565 [parallel-1] INFO - # onNext: 2
14:31:14.570 [parallel-1] INFO - # onNext: 3
// ...
14:31:14.788 [parallel-1] INFO - # onNext: 38
14:31:14.794 [parallel-1] INFO - # onNext: 39
14:31:14.801 [parallel-1] INFO - # onNext: 40
14:31:14.801 [parallel-2] INFO - # dropped: 256
14:31:14.802 [parallel-2] INFO - # dropped: 257
// ...
14:31:14.806 [parallel-2] INFO - # dropped: 261
14:31:14.806 [parallel-1] INFO - # onNext: 41
*/
```

- `onBackpressureDrop()` Operator
  - DROP 전략을 사용
  - Drop 되는 데이터를 파라미터로 전달받을 수 있다.

---

## LATEST 전략

- Publisher가 Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우
  - 버퍼 밖에서 대기 중인 데이터 중에서 가장 최근에 emit된 데이터부터 버퍼에 채운다.
- **새로운 데이터가 들어오는 시점에 가장 최근의 데이터만 남겨 두고 나머지 데이터를 폐기한다.**

```java
Flux
    .interval(Duration.ofMillis(1L))
    .onBackpressureLatest()
    .publishOn(Schedulers.parallel())
    .subscribe(data -> {
            try {
                Thread.sleep(5L);
            } catch (InterruptedException e) {}
            log.info("# onNext: {}", data);
        },
        error -> log.error("# onError", error));

Thread.sleep(2000L);

/*
14:41:14.230 [parallel-1] INFO - # onNext: 0
14:41:14.238 [parallel-1] INFO - # onNext: 1
14:41:14.245 [parallel-1] INFO - # onNext: 2
14:41:14.251 [parallel-1] INFO - # onNext: 3
// ...
14:41:15.844 [parallel-1] INFO - # onNext: 254
14:41:15.850 [parallel-1] INFO - # onNext: 255
14:41:15.857 [parallel-1] INFO - # onNext: 1224
14:41:15.863 [parallel-1] INFO - # onNext: 1225
*/
```

### DROP 전략 vs LATEST 전략

- emit된 데이터를 나 자신이라고 가정하자.
- `DROP 전략`은 나 자신을 폐기하는 것
- `LATEST 전략`은 나 자신 보다 앞에 있는 누군가를 폐기하는 것

---

## BUFFER 전략

1. 버퍼의 데이터를 폐기하지 않고 버퍼링 하는 전략
2. 버퍼가 가득 차면 버퍼 내의 데이터를 폐기하는 전략
3. 버퍼가 가득차면 에러를 발생시키는 전략

### BUFFER DROP_LATEST 전략

- Publisher가 Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우
  - 가장 최근에 버퍼 안에 채워진 데이터를 Drop 한다.

```java
Flux
    .interval(Duration.ofMillis(300L))
    .doOnNext(data -> log.info("# emitted by original Flux: {}", data))
    .onBackpressureBuffer(2,
            dropped -> log.info("** Overflow & Dropped: {} **", dropped),
            BufferOverflowStrategy.DROP_LATEST)
    .doOnNext(data -> log.info("[ # emitted by Buffer: {} ]", data))
    .publishOn(Schedulers.parallel(), false, 1)
    .subscribe(data -> {
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {}
            log.info("# onNext: {}", data);
        },
        error -> log.error("# onError", error));

Thread.sleep(2500L);

/*
15:02:44.771 [parallel-2] INFO - # emitted by original Flux: 0
15:02:44.778 [parallel-2] INFO - [ # emitted by Buffer: 0 ]
15:02:45.074 [parallel-2] INFO - # emitted by original Flux: 1
15:02:45.370 [parallel-2] INFO - # emitted by original Flux: 2
15:02:45.671 [parallel-2] INFO - # emitted by original Flux: 3
15:02:45.674 [parallel-2] INFO - ** Overflow & Dropped: 3 **
15:02:45.784 [parallel-1] INFO - # onNext: 0
15:02:45.784 [parallel-1] INFO - [ # emitted by Buffer: 1 ]
15:02:45.969 [parallel-2] INFO - # emitted by original Flux: 4
15:02:46.269 [parallel-2] INFO - # emitted by original Flux: 5
15:02:46.269 [parallel-2] INFO - ** Overflow & Dropped: 5 **
15:02:46.573 [parallel-2] INFO - # emitted by original Flux: 6
15:02:46.574 [parallel-2] INFO - ** Overflow & Dropped: 6 **
15:02:46.785 [parallel-1] INFO - # onNext: 1
15:02:46.787 [parallel-1] INFO - [ # emitted by Buffer: 2 ]
15:02:46.872 [parallel-2] INFO - # emitted by original Flux: 7
*/
```

- 첫번째 `doOnNext()` Operator를 통해 `interval()` Operator에서 생성된 원본 Flux 데이터가 emit되는 과정을 확인할 수 있다.
- 두번째 `doOnNext()` Operator를 통해 Buffer에서 Downstream으로 emit되는 데이터를 확인할 수 있다.

### BUFFER DROP_OLDEST 전략

- Publisher가 Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우
  - 버퍼 안에 채워진 데이터 중에서 가장 오래된 데이터를 Drop 한다.
- `BUFFER DROP_LATEST 전략`과 정반대

```java
Flux
    .interval(Duration.ofMillis(300L))
    .doOnNext(data -> log.info("# emitted by original Flux: {}", data))
    .onBackpressureBuffer(2,
            dropped -> log.info("** Overflow & Dropped: {} **", dropped),
            BufferOverflowStrategy.DROP_OLDEST)
    .doOnNext(data -> log.info("[ # emitted by Buffer: {} ]", data))
    .publishOn(Schedulers.parallel(), false, 1)
    .subscribe(data -> {
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {}
            log.info("# onNext: {}", data);
        },
        error -> log.error("# onError", error));

Thread.sleep(2500L);

/*
15:14:53.788 [parallel-2] INFO - # emitted by original Flux: 0
15:14:53.795 [parallel-2] INFO - [ # emitted by Buffer: 0 ]
15:14:54.089 [parallel-2] INFO - # emitted by original Flux: 1
15:14:54.383 [parallel-2] INFO - # emitted by original Flux: 2
15:14:54.683 [parallel-2] INFO - # emitted by original Flux: 3
15:14:54.686 [parallel-2] INFO - ** Overflow & Dropped: 1 **
15:14:54.797 [parallel-1] INFO - # onNext: 0
15:14:54.798 [parallel-1] INFO - [ # emitted by Buffer: 2 ]
15:14:54.985 [parallel-2] INFO - # emitted by original Flux: 4
15:14:55.282 [parallel-2] INFO - # emitted by original Flux: 5
15:14:55.284 [parallel-2] INFO - ** Overflow & Dropped: 3 **
15:14:55.586 [parallel-2] INFO - # emitted by original Flux: 6
15:14:55.586 [parallel-2] INFO - ** Overflow & Dropped: 4 **
15:14:55.804 [parallel-1] INFO - # onNext: 2
15:14:55.804 [parallel-1] INFO - [ # emitted by Buffer: 5 ]
15:14:55.881 [parallel-2] INFO - # emitted by original Flux: 7
*/
```
