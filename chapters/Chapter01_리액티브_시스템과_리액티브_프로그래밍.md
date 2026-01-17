# Chapter01 리액티브 시스템과 리액티브 프로그래밍

---

## Reactive System이란?

- reactive
- **반응을 잘하는 시스템**

> 빠른 응답성을 바탕으로 유지보수와 확장이 용이한 시스템

클라이언트의 요청에 즉각적으로 응답함으로써 지연시간을 최소화한다.

---

## 리액티브 선언문으로 리액티브 시스템 이해하기

### Reactive Manifesto

- 리액티브 선언문
- 리액티브 선언문 설계 원칙

#### Means (수단)

- 리액티브 시스템에서 주요 통신 수단으로 무엇을 사용할 것인지 표현한 것
- **비동기 메시지 기반의 통신**

#### Form

- 메시지 기반 통신을 통해서 어떠한 형태를 지니는 시스템으로 형성되는지를 나타낸 것
- 비동기 메시지 통신 기반하에 `탄력성` 과 `회복성` 을 가지는 시스템이어야 한다.
- **Elastic (탄력성)**
  - 시스템의 작업량이 변화하더라도 일정한 응답을 유지하는 것
- **Resilient (회복성)**
  - 시스템에 장애가 발생하더라도 응답성을 유지하는 것

#### Value

- 즉각적으로 응답 가능한 시스템 구축 (`Responsive`)
- 리액티브 시스템의 핵심 가치

---

## 리액티브 프로그래밍이란?

> 리액티브 시스템을 구축하는 데 필요한 프로그래밍 모델

---

## 리액티브 프로그래밍의 특징

[용어집 - 리액티브 선언문](https://www.reactivemanifesto.org/ko/glossary)

> In computing, reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change.

---

## 명령형 프로그래밍 vs 선언형 프로그래밍

- 선언형 프로그래밍에서는 **`동작을 구체적으로 명시하지 않고 목표만 선언한다.`**
- 선언형 프로그래밍에서는 여러 가지 동작을 각각 별도로 코드로 분리하지 않는다.
- 선언형 프로그래밍 방식은 `함수형 프로그래밍`으로 구성된다.

---

## 리액티브 프로그래밍 구성

### Publisher

- 데이터를 제공하는 역할

### Subscriber

- 데이터를 전달받아서 사용하는 역할

### Data Source

- Publisher의 입력으로 들어오는 데이터
- Data Stream

### Operator

- Data Stream의 가공 처리를 담당하는 역할
- Publisher로 부터 전달된 데이터를 가공처리하여 Subscriber에게 전달
