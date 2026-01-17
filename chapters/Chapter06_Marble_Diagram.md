# Chapter06. Marble Diagram

---

## Marble Diagram이란?

- Marble
  - 구슬이라는 의미
  - 구슬 모양으로 표현한 데이터

![Marble Diagram](https://user-images.githubusercontent.com/39241588/128735689-018a61bb-73ea-4d9c-a730-1c99ae6d76da.png)

- Source Flux
  - Operator 함수로 가공 처리되기 전의 데이터 Sequence
- Output Flux
  - Operator 함수로 가공 처리된 후의 데이터 Sequence

---

## Marble Diagram으로 Reactor의 Publisher 이해하기

### Mono 마블 다이어그램

![Mono Marble Diagram](https://velog.velcdn.com/images/bimilless/post/085e0662-d909-45ad-a245-854b10ded707/image.png)

```java
Mono.just("Hello Reactor")
        .subscribe(System.out::println);

// Hello Reactor
```

- RxJava
  - Mono와 비슷한 역할을 하는 Single이라는 Publisher 타입이 존재한다.

```java
Mono
    .empty()
    .subscribe(
            none -> System.out.println("# emitted onNext signal"),
            error -> {},
            () -> System.out.println("# emitted onComplete signal")
    );

// # emitted onComplete signal
```

- `empty()` Operator는 emit할 데이터가 없는 것으로 간주
  - 따라서 onNext Signal이 발생하지 않는다

---

### Flux 마블 다이어그램

![Flux Marble Diagram](https://velog.velcdn.com/images/bimilless/post/ba3c0959-0eb1-4709-a759-18c0ee482d04/image.png)

- emit 되는 구슬 모양의 데이터가 여러 개!

#### Mono

- 0건 또는 1건만 emit 할 수 있는 Publisher 타입

#### Flux

- 여러 건의 데이터를 emit 할 수 있는 Publisher 타입
- Mono의 데이터 emit 범위를 포함한다고 볼 수 있음.
  - 그럼 Mono를 왜 쓸까?

---

### concatWith() 마블 다이어그램

- concatWith 함수
  - 2개의 데이터 소스를 연결하여 하나의 데이터 스트림을 만드는 함수

```java
Flux<String> flux = Mono.justOrEmpty("Steve")
                        .concatWith(Mono.justOrEmpty("Jobs"));

flux.subscribe(System.out::println);

// Steve
// Jobs
```

![concatWith Marble Diagram](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/Single.concatWith.png)

---

### concat()

```java
Flux.concat(
        Flux.just("Mercury", "Venus", "Earth"),
        Flux.just("Mars", "Jupiter", "Saturn"),
        Flux.just("Uranus", "Neptune", "Pluto"))
    // Flux
    .collectList() // Mono
    .subscribe(System.out::println); // List 출력
```

---

## 참고

[ReactiveX - Single](https://reactivex.io/documentation/single.html)
