# Chapter03. Blocking I/O와 Non-Blocking I/O

---

## Blocking vs Non-Blocking

- **호출되는 함수가 바로 리턴하는지 여부가 관심사**
- blocking
  - 호출되는 함수가 자신의 작업을 모두 마칠 때 까지 호출한 함수에게 제어권을 넘겨주지 않고 대기하게 만든다
- non-blocking
  - 호출되는 함수가 바로 리턴해서 호출한 함수에게 제어권을 넘겨주어 다른 일을 할 수 있게 한다

## Sync vs Async

- **호출되는 함수의 작업 완료 여부를 누가 신경쓰냐**가 관심사
- 동기(sync)
  - 호출하는 함수가 호출되는 함수의 작업 완료 여부를 신경쓴다
- 비동기(async)
  - 호출되는 함수에게 callback을 전달해서, 호출되는 함수의 작업이 완료되면 callback을 실행한다
  - 호출하는 함수는 작업 완료 여부를 신경쓰지 않는다

[blocking, non-blocking IO, 동기, 비동기 개념 정리](https://limdongjin.github.io/concepts/blocking-non-blocking-io.html)

---

## Blocking I/O

> 하나의 스레드가 I/O에 의해서 차단되어 대기하는 것

![Blocking I/O](../images/chapter03_blocking_io.png)

### Multi Thread 기법의 문제점

- Context Switching 으로 인한 스레드 전환 비용이 발생
- 과다한 메모리 사용으로 오버헤드가 발생할 수 있다.
  - 일반적으로 JVM에서 스레드 1개당 64KB~1MB 메모리를 차지함
- Thread Pool 에서 응답 지연이 발생할 수 있다.

### 참고

#### Context Switching

> 현재 진행하고 있는 Task(Process, Thread)의 상태를 저장하고 다음 진행할 Task의 상태 값을 읽어 적용하는 과정을 말합니다.

#### PCB

- Process Control Block
- 기존에 실행되고 있는 프로세스의 정보를 저장하는 공간

#### TCB

- Thread Control Block
- 기존에 실행되고 있는 스레드 정보를 저장하는 공간

---

## Non-Blocking I/O

> 작업 스레드의 종료 여부와 관계없이 요청한 스레드는 차단되지 않는다.

![Non-Blocking I/O](../images/chapter03_non_blocking_io.png)

### Non-Blocking I/O 문제점

- 스레드 내부에 **CPU를 많이 사용하는 작업이 포함된 경우에는 성능에 악영향**을 준다.
- 사용자의 요청에서 응답까지의 전체 과정에 Blocking I/O 요소가 포함된 경우에는 Non-Blocking의 이점을 발휘하기가 힘들다.

#### Fully Non-Blocking I/O

- 어느 하나라도 Blocing I/O 요소가 존재한다면
  - Non-Blocking I/O의 이점을 발휘하기가 힘들다

---

## Non-Blocking I/O 방식의 통신이 적합한 시스템

### 1. 대량의 요청 트래픽이 발생하는 시스템

- 서버 증설이나 VM 확장 등을 통해 트래픽을 분산할 수 있다
  - 상대적으로 비용이 많이 든다
- Spring WebFlux 기반 애플리케이션은 상대적으로 저비용으로 고수준의 성능

### 2. 마이크로 서비스 기반 시스템

- 마이크로 서비스 기반의 시스템은 특성상 서비스들 간에 많은 수의 I/O가 지속적으로 발생
- 따라서 특정 서비스들 간의 통신에서 Blocking으로 인한 응답 지연이 발생
- 응답 지연의 연쇄 작용으로 시스템 전체가 마비될 수 있음

### 3. 스트리밍 또는 실시간 시스템

- 무한한 데이터 스트림을 전달받아서 효율적으로 처리 가능
