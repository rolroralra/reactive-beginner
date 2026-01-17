# Chapter02. Reactive Streams

---

## Reactive Streams란?

> 데이터 스트림을 Non-Blocking이면서 비동기적인 방식으로 처리하기 위한 리액티브 라이브러리의 표준 사양

### 리액티브 스트림즈 구현체

- RxJava
- Reactor
- Akka Streams
- Java 9 Flow API

---

## 리액티브 스트림즈 구성요소

| 컴포넌트 | 설명 |
|----------|------|
| Publisher | 데이터를 생성하고 통지하는 역할 |
| Subscriber | 구독한 Publisher로부터 통지된 데이터를 전달받아서 처리하는 역할 |
| Subscription | Publisher에 요청할 데이터의 개수를 지정하고, 데이터의 구독을 취소하는 역할 |
| Processor | Publisher와 Subscriber의 기능을 모두 가지고 있음 |

![리액티브 스트림즈 구성요소](https://velog.velcdn.com/images/korea3611/post/c639afc4-eea1-4c90-abfb-e19676da0b7d/image.png)

### Subscriber가 Subscription.request를 통해 왜 굳이 데이터의 요청 개수를 지정하는 걸까?

- Publisher, Subscriber 실제로는 각각 다른 스레드에서 비동기적으로 상호작용
- 만약 Publisher가 통지하는 속도가 Subscriber가 수신받은 데이터 처리하는 속도보다 빠르다면
- 위 문제를 방지하기 위해, Subscription.request를 통해 데이터 요청 개수를 제어한다.

---

## Publisher

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}
```

- Kafka에서의 Pub/Sub 모델과 리액티브 스트림즈에서의 Pub/Sub 모델은 의미가 조금 다르다.
- Kafka의 경우 Publisher, Subscriber 중간에 Message Broker가 있다.

---

## Subscriber

```java
public interface Subscriber<T> {
    void onSubscribe(Subscription subscription);
    void onNext(T t);
    void onError(Throwable e);
    void onComplete();
}
```

- `onSubscribe`: 구독 시작 시점에 호출
- `onNext`: Publisher가 통지한 데이터를 처리
- `onError`: 에러가 발생했을 때 호출
- `onComplete`: 모든 데이터 통지가 완료되었을 때 호출

---

## Subscription

```java
public interface Subscription {
    void request(long n);
    void cancel();
}
```

- `request(long n)`: n개의 데이터를 요청
- `cancel()`: 구독을 취소

---

## Publisher, Subscriber 동작과정

1. Publisher가 Subscriber 인터페이스 구현 객체를 subscribe 메서드의 파라미터로 전달
2. Publisher 내부에서는 전달받은 Subscriber 인터페이스 구현 객체의 onSubscribe 메서드를 호출
3. 호출된 Subscriber 인터페이스 구현 객체의 onSubscribe 메서드에서 전달받은 Subscription 객체를 통해 전달받을 데이터의 개수를 Publisher에게 요청 (request 메서드)
4. Publisher는 Subscriber로부터 전달받은 요청 개수만큼의 데이터를 onNext 메서드를 호출해서 Subscriber에게 전달
5. Publisher는 통지할 데이터가 더 이상 없을 경우, onComplete 메서드를 호출해서 Subscriber에게 데이터 처리 종료를 알린다.

---

## Processor

```java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> { }
```

- Publisher, Subscriber 기능을 모두 가지고 있음

---

## 리액티브 스트림즈 관련 용어

### Signal

- Publisher와 Subscriber간에 주고받는 상호작용
- Publisher가 Subscriber에게 보내는 Signal
- Subscriber가 Publisher에게 보내는 Signal

### Demand

- Subscriber가 Publisher에게 요청하는 데이터를 의미
- Publisher가 아직 Subscriber에게 전달하지 않은 데이터

### Emit

- 데이터를 내보내다.
- 통지, 발행, 게시, 방출

### Upstream, Downstream

- Upstream: 현재 Operator를 기준으로 상위에 있는 Operator
- Downstream: 현재 Operator를 기준으로 하위에 있는 Operator

### Sequence

- 다양한 Operator로 데이터의 연속적인 흐름

### Operator

- just
- filter
- map
- etc…

### Source

- 최초에 가장 먼저 생성된 무언가

---

## 리액티브 스트림즈의 구현 규칙

> https://github.com/reactive-streams/reactive-streams-jvm

### Publisher 규칙

1. Publisher가 Subscriber의 onNext 메소드를 호출한 총 횟수는 Subscriber가 Subscription을 통해 요청한 총 수보다 작거나 같아야 한다.
2. Publisher는 요청받은 것보다 더 적게 onNext를 호출할 수 있다. 그리고 onComplete 또는 onError를 호출해서 Subscription를 종료해야한다.
3. Publisher는 실패시에 반드시 onError를 호출해야한다.
4. Publisher가 성공적으로 완료된 경우에는 onComplete를 호출해야 한다.
5. Publisher가 onComplete 혹은 onError를 호출한 경우, Subscriber의 Subscription은 취소된 것으로 간주되어야 한다.
6. onError, onComplete 메서드가 한번 호출됬다면, 더이상 호출하면 안된다.
7. Subscription 객체의 cancel 메서드를 통해 Subscription이 취소된 경우, Subscriber에 추가적인 호출을 하면 안된다.

### Subscriber 규칙

1. Subscription.request(n) 메서드를 통해 데이터를 요청해야 한다. 그리고 데이터는 onNext 메소드를 통해 받는다.
2. Subscriber.onComplete() 메소드나 onError() 메서드에서는 Subscription 또는 Publisher의 어떠한 메서드도 호출해서는 안된다.
3. Subscriber.onComplete(), onError() 메소드가 호출되면 Subscription이 취소된 것으로 간주해야 한다.
4. Subscription이 더이상 필요하지 않다면, 반드시 Subscription.cancel()를 호출해야 한다.
5. Subscriber.onSubscribe 메서드는 한번만 호출되어야 한다.

### Subscription 규칙

1. Subscriber는 onNext 또는 onSubscribe 메소드 내에서 Subscription.request를 동기적으로 호출할 수 있다.
2. Subscription이 취소된 이후에 추가적인 Subscription.request(long n)은 비작동(non-operation)해야한다.
3. Subscription이 취소된 이후에 추가적인 Subscription.cancel()은 비작동(non-operation)해야한다.
4. Subscription.request(long n) 메소드는 전달된 파라미터가 0보다 작거나 같은 경우 IllegalArgumentException으로 onError를 호출해야한다.
5. Subscription.cancel()은 Publisher에게 Subscriber 메서드 호출을 중단하도록 요청해야한다.
6. Subscription.cancel()은 결국 해당 Subscriber에 대한 참조를 삭제하도록 Publisher에 요청해야한다.
7. Subscription.cancel, Subscription.request 호출에 대한 응답으로 예외를 던지는 것을 허용하지 않는다.
8. Subscription의 request 메서드는 무한대로 호출될 수 있어야 한다.

---

## 리액티브 스트림즈 구현체

### RxJava

- Reactive Extensions
- .NET 환경의 리액티브 확장 라이브러리를 넷플릭스에서 Java 언어로 포팅하여 만든 JVM 기반의 대표적인 리액티브 확장 라이브러리
- 2.0부터 리액티브 스트림즈 사양을 지원
- 1.x 버전과 2.0 이후 버전의 차이점

### Reactor

- Spring Framework 팀에 의해 주도적으로 개발된 리액티브 스트림즈의 구현체

### Akka Streams

- JVM상에서의 동시성과 분산 애플리케이션을 단순화해 주는 오픈소스 Toolkit
- Actor Model을 적극적으로 사용하는 대표적인 기술
- Actor들 간의 통신은 메시지를 통해서만 이루어짐
- Actor들은 서로 독립적
