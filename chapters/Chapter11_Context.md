# Chapter11. Context

---

## Context란?

> Context는 어떠한 상황에서 그 상황을 처리하기 위해 필요한 정보

### Context 예시

- `ServletContext`는 Servlet이 Servlet Container와 통신하기 위해서 필요한 정보를 제공하는 인터페이스
- Spring Framework에서 `ApplicationContext` 는 애플리케이션의 정보를 제공하는 인터페이스
- Spring Security에서 SecurityContextHolder가 관리하는, `SecurityContext` 는 애플리케이션 사용자의 인증 정보를 제공하는 인터페이스

---

## Reactor에서의 Context란?

- Operator 같은 Reactor 구성요소 간에 전파되는 Key / Value 형태의 저장소
- Operator 체인상의 각 Operator가 해당 Context의 정보를 동일하게 이용할 수 있음
- ThreadLocal과 다소 유사한 면이 있지만,
  - 실행 스레드와 매핑되는 것이 아니라, Subscriber와 매핑된다.
- **구독이 발생할 때마다 해당 구독과 연결된 하나의 Context가 생긴다.**

> Reactor에서는 Operator 체인상의 서로 다른 스레드들이 Context에 저장된 데이터에 손쉽게 접근할 수 있다.

### 코드 예시

```java
Mono
    .deferContextual(ctx ->
        Mono
            .just("Hello" + " " + ctx.get("firstName"))
            .doOnNext(data -> log.info("# just doOnNext : {}", data))
    )
    .subscribeOn(Schedulers.boundedElastic())
    .publishOn(Schedulers.parallel())
    .transformDeferredContextual(
            (mono, ctx) -> mono.map(data -> data + " " + ctx.get("lastName"))
    )
    .contextWrite(context -> context.put("lastName", "Jobs"))
    .contextWrite(context -> context.put("firstName", "Steve"))
    .subscribe(data -> log.info("# onNext: {}", data));

Thread.sleep(100L);

/*
19:58:52.925 [boundedElastic-1] INFO - # just doOnNext : Hello Steve
19:58:52.934 [parallel-1] INFO - # onNext: Hello Steve Jobs
*/
```

- `deferContextual()`
  - Context에 저장된 데이터와 원본 데이터 소스의 처리를 지연시키는 역할
- `transformDeferredContextual()` Operator를 사용해서 Operator 체인 중간에서 데이터를 읽을 수 있다.
- `contextWrite()` Operator를 사용해서 Context에 데이터 쓰기 작업을 할 수 있다.

---

## ContextView

- Context에 저장된 데이터를 읽을 때만 사용
- `context.readOnly()`

---

## Context의 특징

- **구독이 발생할 때마다 해당하는 하나의 Context가 하나의 구독에 연결된다.**
  - 구독이 발생할 때마다 Context는 독립적으로 생성된다.
- Context는 Operator 체인 아래에서 위로 전파된다.
- 동일한 키에 대한 값을 중복해서 저장하면, Operator 체인에서 가장 위쪽에 위치한 `contextWrite()` Operator가 저장한 값으로 덮어 쓴다.
- `contextWrite()` 는 Operator 체인의 맨 마지막에 둡니다!
- Inner Sequence 내부에서는 외부 Context에 저장된 데이터를 읽을 수 있다.
- Inner Sequence 외부에서는 Inner Sequence 내부 Context에 저장된 데이터를 읽을 수 없다.
