# Chapter04. 리액티브 프로그래밍을 위한 사전 지식

---

## 함수형 인터페이스 (Functional Interface)

> 단 하나의 추상 메서드만 있는 인터페이스

```java
@FuntionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);
    // ...
}
```

---

## 람다 표현식 (Lambda Expression)

```java
(String a, String b) -> a.equals(b)
```

> 함수형 인터페이스를 구현한 클래스의 인스턴스를 람다 표현식으로 작성해서 전달한다.

### 람다 캡처링

- 람다 표현식 외부에서 정의된 자유 변수를 람다 표현식에 사용하는 것!

---

## 메서드 레퍼런스 (Method Reference)

```java
(Car car) -> car.getCarName()

Car::getCarName
```

- ClassName :: static method
- ClassName :: instance method
- object :: instance method
- ClassName :: new

---

## 함수 디스크립터 (Function Descriptor)

| 함수형 인터페이스 | 함수 디스크립터 |
|------------------|----------------|
| Predicate<T> | T -> boolean |
| Consumer<T> | T -> void |
| Function<T, R> | T -> R |
| Supplier<T> | () -> T |
| BiPredicate<L, R> | (L, R) -> boolean |
| BiConsumer<T, U> | (T, U) -> void |
| BiFunction<T, U, R> | (T, U) -> R |
