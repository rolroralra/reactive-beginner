# Chapter07. Cold Sequence와 Hot Sequence

---

## Cold와 Hot의 의미

> Cold는 무언가를 새로 시작하고, Hot은 무언가를 새로 시작하지 않는다.

---

## Cold Sequence

> Subscriber가 구독할 때 마다 데이터 흐름이 처음부터 다시 시작되는 Sequence

![Cold Sequence](https://cdn.inflearn.com/public/files/courses/325963/b5d34a8a-f66c-4bdc-8230-3c82f207a21f/K-008.png)

```java
Flux<String> coldFlux =
    Flux
        .fromIterable(Arrays.asList("KOREA", "JAPAN", "CHINESE"))
        .map(String::toLowerCase);

coldFlux.subscribe(country -> log.info("# Subscriber1: {}", country));

System.out.println("----------------------------------------------------------------------");

Thread.sleep(2000L);

coldFlux.subscribe(country -> log.info("# Subscriber2: {}", country));

// 01:27:07.332 [main] DEBUG- Using Slf4j logging framework
// 01:27:07.340 [main] INFO - # Subscriber1: korea
// 01:27:07.340 [main] INFO - # Subscriber1: japan
// 01:27:07.340 [main] INFO - # Subscriber1: chinese
// ----------------------------------------------------------------------
// 01:27:09.347 [main] INFO - # Subscriber2: korea
// 01:27:09.348 [main] INFO - # Subscriber2: japan
// 01:27:09.352 [main] INFO - # Subscriber2: chinese
```

---

## Hot Sequence

> 구독이 발생한 시점 이전에 Publisher로부터 emit 된 데이터는 Subscriber가 전달받지 못한다.
> 구독이 발생한 시점 이후에 emit된 데이터만 전달받을 수 있다.

![Hot Sequence](https://cdn.inflearn.com/public/files/courses/325883/5cc9cbd7-158b-47ab-90f9-5d0001e8a259/K-006.png)

```java
String[] singers = {"Singer A", "Singer B", "Singer C", "Singer D", "Singer E"};

log.info("# Begin concert:");

Flux<String> concertFlux =
        Flux
            .fromArray(singers)
            .delayElements(Duration.ofSeconds(1))
            .share();

concertFlux.subscribe(
        singer -> log.info("# Subscriber1 is watching {}'s song", singer));

Thread.sleep(2500);

concertFlux.subscribe(
        singer -> log.info("# Subscriber2 is watching {}'s song", singer));

Thread.sleep(3000);

// 01:31:24.967 [main] INFO - # Begin concert:
// 01:31:25.018 [main] DEBUG- Using Slf4j logging framework
// 01:31:26.064 [parallel-1] INFO - # Subscriber1 is watching Singer A's song
// 01:31:27.070 [parallel-2] INFO - # Subscriber1 is watching Singer B's song
// 01:31:28.080 [parallel-3] INFO - # Subscriber1 is watching Singer C's song
// 01:31:28.081 [parallel-3] INFO - # Subscriber2 is watching Singer C's song
// 01:31:29.087 [parallel-4] INFO - # Subscriber1 is watching Singer D's song
// 01:31:29.091 [parallel-4] INFO - # Subscriber2 is watching Singer D's song
// 01:31:30.101 [parallel-5] INFO - # Subscriber1 is watching Singer E's song
// 01:31:30.101 [parallel-5] INFO - # Subscriber2 is watching Singer E's song
```

- `delayElements()`
  - 데이터를 emit하는 시간을 지연시킨다
- `share()`
  - Cold Sequence를 Hot Sequence로 변환해주는 Operator
  - 여러 Subscriber가 하나의 원본 Flux를 공유한다

---

### cache() Operator

- Cold Sequence로 동작하는 Publisher를 Hot Sequence로 변경해준다.
- emit된 데이터를 캐시한 뒤, 구독이 발생할 때 마다 캐시된 데이터를 전달한다.

```java
public static void main(String[] args) throws InterruptedException {
    URI worldTimeUri = UriComponentsBuilder.newInstance().scheme("http")
            .host("worldtimeapi.org")
            .port(80)
            .path("/api/timezone/Asia/Seoul")
            .build()
            .encode()
            .toUri();

    Mono<String> mono = getWorldTime(worldTimeUri).cache();

    mono.subscribe(dateTime -> log.info("# dateTime 1: {}", dateTime));
    Thread.sleep(2000);
    mono.subscribe(dateTime -> log.info("# dateTime 2: {}", dateTime));

    Thread.sleep(2000);
}

private static Mono<String> getWorldTime(URI worldTimeUri) {
    return WebClient.create()
            .get()
            .uri(worldTimeUri)
            .retrieve()
            .bodyToMono(String.class)
            .map(response -> {
                DocumentContext jsonContext = JsonPath.parse(response);
                return jsonContext.read("$.datetime");
            });
}
```

---

## 정리

- Subscriber의 구독 시점이 달라도, 구독할 때마다 Publisher가 데이터를 처음부터 emit하는 과정을 `Cold Sequence`라고 한다.
- `Cold Sequence` 흐름으로 동작하는 Publisher를 `Cold Publisher`라고 한다.
- Publisher가 데이터를 emit하는 과정이 한 번만 일어나고, Subscriber가 각각의 구독 시점 이후에 emit된 데이터만 전달받는 것을 `Hot Sequence`라고 한다.
- `Hot Sequence` 흐름으로 동작하는 Publisher를 `Hot Publisher` 라고 한다.
- share(), cache() 등의 Operator를 사용해서 Cold Sequence를 Hot Seqeunce로 변환할 수 있다.
