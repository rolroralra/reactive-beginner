# Reactive Beginner

Spring WebFlux와 Reactive Streams를 학습하기 위한 문서 모음입니다.

## 목차

### 기초 개념
| Chapter | 제목 |
|---------|------|
| 01 | [리액티브 시스템과 리액티브 프로그래밍](chapters/Chapter01_리액티브_시스템과_리액티브_프로그래밍.md) |
| 02 | [Reactive Streams](chapters/Chapter02_Reactive_Streams.md) |
| 03 | [Blocking I/O와 Non-Blocking I/O](chapters/Chapter03_Blocking_IO와_Non-Blocking_IO.md) |
| 04 | [리액티브 프로그래밍을 위한 사전 지식](chapters/Chapter04_리액티브_프로그래밍을_위한_사전_지식.md) |

### Reactor 핵심
| Chapter | 제목 |
|---------|------|
| 05 | [Reactor 개요](chapters/Chapter05_Reactor_개요.md) |
| 06 | [Marble Diagram](chapters/Chapter06_Marble_Diagram.md) |
| 07 | [Cold Sequence와 Hot Sequence](chapters/Chapter07_Cold_Sequence와_Hot_Sequence.md) |
| 08 | [Backpressure](chapters/Chapter08_Backpressure.md) |
| 09 | [Sinks](chapters/Chapter09_Sinks.md) |
| 10 | [Scheduler](chapters/Chapter10_Scheduler.md) |
| 11 | [Context](chapters/Chapter11_Context.md) |

### 테스트와 디버깅
| Chapter | 제목 |
|---------|------|
| 12 | [Debugging](chapters/Chapter12_Debugging.md) |
| 13 | [Testing](chapters/Chapter13_Testing.md) |

### Operators
| Chapter | 제목 |
|---------|------|
| 14 | [Operators 개요](chapters/Chapter14_Operators.md) |

#### Operator 상세
| 문서 | 설명 |
|------|------|
| [Sequence 생성](chapters/operators/Operator_Sequence_생성.md) | just, fromIterable, fromStream, range, generate, create 등 |
| [Sequence 필터링](chapters/operators/Operator_Sequence_필터링.md) | filter, skip, take, next 등 |
| [Sequence 변환](chapters/operators/Operator_Sequence_변환.md) | map, flatMap, concat, merge, zip 등 |
| [내부 동작](chapters/operators/Operator_Sequence_내부동작.md) | doOnSubscribe, doOnNext, doOnError, doOnComplete 등 |
| [에러 처리](chapters/operators/Operator_에러처리.md) | error, onErrorReturn, onErrorResume, onErrorContinue, retry |
| [시간 측정](chapters/operators/Operator_시간측정.md) | elapsed |
| [Flux 분할](chapters/operators/Operator_Flux_분할.md) | window, buffer, groupBy |
| [Multicasting](chapters/operators/Operator_Multicasting.md) | publish, autoConnect, refCount |

---

## Spring WebFlux 가이드

Spring WebFlux를 활용한 실전 웹 애플리케이션 개발 가이드입니다.

### 기본
| 문서 | 설명 |
|------|------|
| [Spring WebFlux 개요](spring-webflux/01_Spring_WebFlux_개요.md) | WebFlux 소개, MVC와의 비교 |
| [시작하기](spring-webflux/02_시작하기.md) | 프로젝트 설정, 첫 번째 애플리케이션 |
| [Annotated Controllers](spring-webflux/03_Annotated_Controllers.md) | 어노테이션 기반 컨트롤러 |
| [Functional Endpoints](spring-webflux/04_Functional_Endpoints.md) | 함수형 엔드포인트 |

### 통신
| 문서 | 설명 |
|------|------|
| [WebClient](spring-webflux/05_WebClient.md) | 리액티브 HTTP 클라이언트 |
| [WebSocket](spring-webflux/06_WebSocket.md) | 양방향 실시간 통신 |
| [Server-Sent Events](spring-webflux/07_Server_Sent_Events.md) | 서버 푸시 이벤트 |

### 고급
| 문서 | 설명 |
|------|------|
| [예외 처리](spring-webflux/08_예외_처리.md) | 에러 핸들링 전략 |
| [Filter와 Interceptor](spring-webflux/09_Filter와_Interceptor.md) | 요청/응답 필터링 |
| [Testing](spring-webflux/10_Testing.md) | WebTestClient, StepVerifier |
| [Spring Security 통합](spring-webflux/11_Spring_Security_통합.md) | 인증, 인가, JWT |
| [R2DBC](spring-webflux/12_R2DBC.md) | 리액티브 데이터베이스 연결 |

---

## Marble Diagram

각 Operator의 동작 방식을 시각적으로 이해할 수 있도록 [Project Reactor](https://projectreactor.io/) 공식 문서의 Marble Diagram을 포함하고 있습니다.

![map Operator](images/mapForFlux.svg)

## 참고 자료

- [Project Reactor 공식 문서](https://projectreactor.io/docs/core/release/reference/)
- [Spring WebFlux 공식 문서](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Reactive Streams 스펙](https://www.reactive-streams.org/)
