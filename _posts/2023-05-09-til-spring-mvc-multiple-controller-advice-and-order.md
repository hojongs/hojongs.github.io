---
layout: post
title: "[TIL] Spring MVC: Multiple ControllerAdvices & Order"
categories: [Spring MVC]
tags: [Spring, Spring MVC, TIL, Exception handling]
---

## ControllerAdvice란? 사용 이유?

Controller 이하 level에서 핸들링되지 않은 예외를 핸들링하기 위해 `@ControllerAdvice`를 사용할 수 있다.

IOException과 같은 예외는 보통 이런 방식으로 처리하는 것이 유용하다.

즉, ControllAdvice는 전역적 에러 핸들링 코드 구현을 편리하게 만들어준다. 새로운 Controller 클래스에도 별도 설정없이 기본적으로 적용된다.

> 좀더 정확히는, ControllerAdvice는 Controller에 Advice(in terms of AOP)를 추가하는 것이고 예외 핸들링 코드는 ExceptionHandler에 의해 연결된다.
> 
> 전역 예외 핸들링이 ControllerAdvice의 가장 많은 use case일 것이다.

```kotlin
@ControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(Exception::class)
    fun handleAllExceptions(ex: Exception, request: javax.servlet.http.HttpServletRequest): ResponseEntity<*> {
        // ...
    }
}
```

주로 Spring MVC에서 많이 사용되는 기능인데, WebFlux 서버에서 Controller를 사용한다면 동일하게 적용 가능하다.
단, `HttpServletRequest` 대신 `org.springframework.http.server.reactive.ServerHttpRequest`를 사용해야 할 것이다.

## Multiple ControllerAdvices

ControllerAdvice 클래스가 여러 개 있을 경우에는 어떻게 동작할까?

우선 이렇게 사용할 필요가 있는지부터 검토해보는 것이 좋다.

- 각 ControllerAdvice가 담당하는 Controller들이 다를 경우: annotation value를 통해 설정 가능하며 이 경우는 당연히 유용하다. 궁금한 것은 겹치는 Controller가 있을 때이다.
- 하나의 Controller에게 여러 개의 ControllerAdvice가 매핑되는 경우: 이 경우에 대해 살펴보자.

### Use case 1: 각 수준의 exception들을 핸들링하는 ControllerAdvice 분리

에외는 다양한 수준에서 발생할 수 있다.

- 외부 서비스 HTTP Call에서 예외가 발생
- 파일을 읽거나 네트워크 통신 중 IOException 발생
- 서비스 로직에서 예외 발생
- Controller에서 request parameter validation 등 예외 발생

이러한 예외들을 하나의 ControllerAdvice에서 처리하도록 작성할 수도 있지만 예외들을 수준별로 분류하여 여러 개의 ControllerAdvice에서 처리하도록 구성하는 것이 더 유용할 수도 있다.

### Use case 2: 공통 예외 핸들링 로직과 응용 특화 예외 핸들링 로직

Use case 1과 비슷하지만, 공통 예외와 응용 특화 예외를 여러 개의 ControllerAdvice로 분류하여 처리하는 것이 더 유용할 수도 있다.

그렇지 않으면 하나의 ControllerAdvice가 수많은 클래스에 의존하게 된다. 즉 의존성이 많아지고 이는 클래스를 더 복잡하게 만들 수 있다.

의존하는 클래스가 다른 모듈에 있으면 ControllerAdvice의 모듈은 그 모듈들을 의존성에 추가해야 한다. 이런 경우 꽤 복잡한 문제가 될 수 있다. (해결방법은 여러 가지일 것이다)

단일 ControllerAdvice로 구현하는 것도 좋은 방법이므로 오해하지 말자. 특정 예외가 핸들링된 위치를 찾기 위해 여러 개의 ControllerAdvice 클래스를 헤매는 단점이 있을 수도 있다. (결국 디자인 결정의 문제)

결국 관심사의 분리(separation of concerns)라는 디자인 원칙을 달성하기 위한 접근하기 위한 방법이라고 볼 수 있다.

<https://en.wikipedia.org/wiki/Separation_of_concerns>

## Implementation

사실 잘 생각해보면, 하나의 Controller에 대해 여러 개의 ControllerAdvice의 동작을 이해하는 것이 어렵지 않다.

**이름에서 알 수 있듯이 ControllerAdvice는 결국 Advice이고, Advice는 Order를 통해 순서를 지정할 수 있다.** 이 규칙은 ControllerAdvice에도 동일하게 적용된다.

Order 어노테이션을 사용하거나 Ordered interface를 구현하면 된다. 숫자가 작은 것이 더 높은 우선순위를 가지므로 헷갈리지 말자.

```kotlin
@ControllerAdvice
@Order(1)
class HighPriorityExceptionHandler {

    @ExceptionHandler(NullPointerException::class)
    fun handleNullPointerException(ex: Exception, request: HttpServletRequest): ResponseEntity<*> {
        // ...
    }
}

@ControllerAdvice
@Order(2)
class LowPriorityExceptionHandler {

    @ExceptionHandler(Exception::class)
    fun handleAllExceptions(ex: Exception, request: HttpServletRequest): ResponseEntity<*> {
        // ...
    }
}
```

## Reference

ref: <https://stackoverflow.com/a/19500823/12956829>
