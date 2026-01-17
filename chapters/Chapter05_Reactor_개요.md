# Chapter05. Reactor 개요

---

## Reactor란?

### 1. Reactive Streams

- Reactive Streams 사양을 구현한 리액티브 라이브러리

### 2. Non-Blocking

- JVM 위에서 실행되는 Non-Blocking Application

### 3. Java's Functional API

- Publisher, Subscriber 간의 상호작용은 Java의 함수형 프로그래밍 API를 통해서 이루어 진다.

### 4. Flux[N]

- Reactor의 Publisher 타입은 크게 2가지
  - Flux: 0~N개의 데이터를 emit
  - Mono: 0~1개의 데이터를 emit

### 5. Mono[0|1]

- 데이터를 한 건도 emit하지 않거나 단 한 건만 emit하는 단발성 데이터 emit에 특화된 Publisher

### 6. Well-suited for microservices

- 마이크로 서비스 기반 시스템에 적합

### 7. BackPressure-ready Network

- Reactor는 Publisher로부터 전달받은 데이터를 처리하는 데 있어 과부하가 걸리지 않도록 제어하는 `Backpressure` 를 지원한다.
- Publisher로부터 전달되는 대량의 데이터를 Subscriber가 적절하게 처리하기 위한 제어 방법

---

## Hello Reactor

```java
Flux<String> sequence = Flux.just("Hello", "Reactor");

sequence.map(String::toLowerCase)
        .subscribe(System.out::println);
```

- just
- map
- subscribe
- 모두 Operator
