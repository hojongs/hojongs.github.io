---
layout: post
title: "Kotlin에서 Kotlin답게 에러 처리하는 방법 (Advanced Error Handling in Kotlin)"
categories: [Kotlin]
tags: [Kotlin, Error handling, Functional programming, Coroutines, Asynchronous programming, Parellel programming, Kotlin sealed class]
---

많은 Kotlin 코드들이 Exception(예외)를 통해 에러를 출력하고 핸들링하고 있을지도 모른다. 대부분은 Java-style 코딩 방식에서 그대로 온 것이라고 추측한다. Kotlin은 에러 헨들링을 위해 Exception 외의 인터페이스들도 제시한다.

본 포스트에서는 Kotlin에서 간결하게 에러 핸들링 코드를 작성하는 방법을 설명한다.

> 본 포스트는 아래 문서를 기반으로 작성하였다.  
> <https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md>

## 가장 익숙한 에러 헨들링 방식 (throw Exception)

실패(에러)를 처리할 때 우리에게 가장 익숙한 방법은 `throw Exception`, 즉 예외이다.

예외는 본질적으로 순차적(sequential)이고, 순차적 코드에서 첫 번째 예외가 발생하면 이후의 코드는 실행되지 않고 호출자(caller)에게 예외가 전파된다. 순차적으로 코드를 작성하는 하나의 coroutine에서도 예외는 잘 작동한다.

하지만 예외는 아래와 같은 유즈 케이스들에서는 코드를 더 복잡하게 만든다.

- **병렬 프로그래밍 (e.g. multiple coroutines)**
- **이후 처리를 위해 여러 개의 실패를 유지해야 할 때**

**이를 위해 Kotlin stdlib에 Result 타입이 추가되었다.** Result는 성공 결과(T) 또는 실패 결과(Throwable) 둘 중 하나의 값을 저장한다. 어떻게 위와 같은 유즈 케이스들을 간단하게 구현할 수 있는걸까?

> NOTE: Result 타입은 Kotlin 1.5부터 함수의 return type으로 직접 선언할 수 있다.

## Use cases

아래는 Result type이 유용한 유즈 케이스들(사례들)이다.

### Continuation and similar callbacks

Result 타입은 원래 Continuation interface 구현을 돕기 위해 제안되었다. Continuation은 Kotlin 내부적으로 coroutine 구현에 사용된다. 하지만 이 포스트에서는 Continuation에 대해서는 자세히 다루지 않을 것이다.

### Asynchronous parallel decomposition

[참고 문서](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#asynchronous-parallel-decomposition)에 나와있는 예제 코드를 보자.

```kotlin
val deferreds: List<Deferred<T>> = List(n) { 
    async { 
        /* Do something that produces T or fails */ 
    } 
}
val outcomes1: List<T> = deferreds.map { it.await() } // BAD -- crash on the first (by index) failure
val outcomes2: List<T> = deferreds.awaitAll() // BAD -- crash on the earliest (by time) failure 
val outcomes3: List<Result<T>> = deferreds.map { runCatching { it.await() } } // !!! <= THIS IS THE ONE WE WANT
```

위에서 "모든" 비동기 병렬 호출(Deferred)의 결과(success or failure)를 받은 후 처리하려면 어떻게 해야할까? 위 3가지의 결과는 각각 아래와 같다.

- outcome1: n번째 원소에서 실패가 발생할 때 crash가 발생한다. 즉, 이후의 결과들은 받을 수 없다.
- outcome2: 가장 먼저 실패가 발생할 때 crash가 발생한다.
- outcome3: **crash 없이 모든 호출 결과를 얻을 수 있다! (실패가 포함되어 있더라도)**

### Functional bulk manipulation of failures

**Kotlin은 함수형 스타일 코드 작성을 권장한다.** 함수형 코드는 업무(business) 관련 실패가 nullable type 또는 sealed class 구조로 표현되면 잘 동작한다. 하지만 로컬 핸들링을 요구하지 않는 exception으로 표현되는 다른 종류의 실패들이 발생하는 경우에는 그렇지 않다.

예외에 강하게 의존하는 Java-style API들을 마주하거나 예외를 로컬에서 핸들링해야하는 경우에는 Kotlin primitive(nullable, sealed class)로는 부족하다. 아래 예시를 보자.

> 로컬 핸들링: 호출자(caller site)에서 예외를 처리하는 것

[참고 문서](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#functional-bulk-manipulation-of-failures)에서는 아래와 같은 함수를 예시로 들었다.

```kotlin
fun readFileData(file: File): Data
```

위 함수에서는 이러한 예외가 발생할지도 모른다.

- 파일이 존재하지 않음
- 파일의 일부가 잘렸음
- 파일을 읽는도중(File I/O) interrupt가 발생했음
- 피일 인코딩이나 포맷에 문제가 있음

이러한 경우에 위 함수는 예외를 던질 것이다. 이제 위 함수를 이용해서 여러 개의 파일을 읽어서 결과를 반환하는 `readFiles` 함수를 구현한다고 해보자. **단, 일부 파일을 읽는 것이 실패하더라도 나머지 파일들을 읽은 결과들도 함께 반환해야 한다.**

기본적으로 `readFileData` 함수가 예외를 던지면 프로그램 진행은 멈추고 나머지 파일들은 읽지 않을 것이다. 따라서 이러한 예외들을 catch하여 나머지 파일들을 읽은 결과도 반환 결과의 포함할 수 있도록 해야 한다. 더 나아가, `readFiles` 함수는 아래와 같이 함수형 스타일로 구현될 수 있다.

```kotlin
fun readFilesCatching(files: List<File>): List<Result<Data>> =
    files.map { 
        runCatching { 
            readFileData(it)
        }
    }
```

> Catching이라는 suffix는 호출자(caller)에게 예외들은 모두 catch 되었으며 이러한 실패 결과들을 처리하는 것은 호출자의 책임이라는 것을 명시적으로 알린다.

runCatching 함수를 통해 예외를 잡아서 Result 타입으로 캡슐화하였다. 이제 실패 결과들을 유지하면서 `readFilesCatching`의 결과값들을 변환하는 코드를 “함수형 스타일로” 표현할 수 있다.

```kotlin
readFilesCatching(files).map { result: Result<Data> -> // type explicitly written here for clarity
    result.map { it.doSomething() } // Operates on Success case, while preserving Failure
}
```

위 코드의 doSomething() 함수에서도 마찬가지로 예외를 던질지도 모른다. 이 때는 mapCatching 메소드를 사용하여 예외를 catch하여 여전히 실패 결과값들을 유지할 수 있다.

```kotlin
readFilesCatching(files).map { result: Result<Data> -> 
    result.mapCatching { it.doSomething() }
}
```

### Functional error handling

**대부분의 함수형 코드에서 try-catch 문은 잘 어울리지 않는다.** 아래의 RxKotlin, Java CompletableFuture 코드를 보자. 모든 코드 스니펫은 성공일 경우 processData 함수를, 실패일 경우 showErrorDialog 함수를 호출한다.

RxKotlin

```kotlin
doSomethingAsync()
    .subscribe(
        { processData(it) },
        { showErrorDialog(it) }
    )
```

Java CompletableFuture

```kotlin
doSomethingAsync()
    .whenComplete { data, exception ->
        if (exception != null) 
            showErrorDialog(exception)
        else 
            processData(data)
    }
```

그리고 마지막으로 try-catch 문이다. 위 코드들과 다르게 동기(synchronous) 함수를 사용하고 있다.

```kotlin
try { 
    val data = doSomethingSync()
    processData(data) 
} catch(e: Throwable) { 
    showErrorDialog(e) 
}
```

**사실 이 코드는 위의 있는 코드들과 다른 의미(semantics)를 가진다.** processData 함수에서 발생한 예외까지 catch된다는 차이점 때문이다. try-catch문을 이용하여 함수형 스타일의 에러 핸들링 semantics를 유지하는 것은 간단하지 않다(non-trivial). 대안에 대해서는 아래 섹션에서 다룬다.

try-catch 문 대신 runCatching 함수를 사용하면 아래와 같이 좀더 함수형 스타일로 작성할 수 있다.

```kotlin
runCatching { doSomethingSync() }
    .onFailure { showErrorDialog(it) }
    .onSuccess { processData(it) }
```

## Alternatives

Kotlin 커뮤니티의 여러 가지 라이브러리들이 success or failure union type을 제공한다. 하지만 표준 라이브러리에서 이러한 라이브러리들에 의존할 수는 없다. 이 섹션에서는 이러한 의존없이 작성된 코드들에 대해서 살펴본다.

### Error handling alternative

성공 결과값의 nullable 여부에 따라 아래와 같은 대안 코드들이 있다. [참고 문서](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#error-handling-alternative)의 코드를 그대로 인용했다.

Non-nullable일 경우

```kotlin
val data: Data? = try { 
        doSomethingSync() 
    } catch(e: Throwable) { 
        showErrorDialog(e)
        null 
    }
if (data != null)    
    processData(data)
```

Nullable일 경우

```kotlin
var data: Data? = null
val success = try { 
        data = doSomethingSync()
        true 
    } catch(e: Throwable) { 
        showErrorDialog(e)
        false 
    }
if (success)    
    processData(data)
```

runCatching 함수를 이용한 코드와 비교했을 때, 상대적으로 코드가 많이 길어진다. (가독성이 낮아보인다) 또한 nullable 여부에 따라 2가지 코드로 나뉘어야 한다는 점과 mutable 변수(var)에 의존해야 한다는 점이 아쉽다.

## Error-handling style and exceptions

> 문서를 읽으면서 이 부분이 가장 흥미로웠다.

Result 클래스는 일반적인 실패를 캡처하도록 설계되었다. Result 클래스는 도메인별 에러 상황을 표현하기 위해 설계되지 않았다.

일반적으로 API가 호출자(caller)에게 실패를 locally(호출지에서) 처리하기를 요구하는(호출 직후 또는 근처에서) 케이스는 아래 2가지로 분류된다.

- nullable 타입 사용: **해당 실패가 추가적인 업무(business) 의미를 포함하지 않을 경우**
- 도메인 특화 데이터 타입 사용 (sealed class): 이 타입은 성공 또는 실패 결과를 표현한다. **실패는 업무 관련 데이터를 포함하여 처리(handling) 시에 활용할 수 있도록 한다.**

문서에서는 아래 API를 예시로 들었다:

```kotlin
fun findUserByName(name: String): Result<User> // ERROR
```

처리하기를 원하는 실패가 오직 해당하는 User를 찾지 못하는 것이라면, 아래와 같은 시그니처를 사용할 수 있다:

```kotlin
fun findUserByName(name: String): User? // Ok
```

위와 달리 여러 종류의 실패를 각각 다르게 처리해야 한다면, 아래와 같은 시그니처를 사용할 수 있다:

```kotlin
sealed class FindUserResult {
    data class Found(val user: User) : FindUserResult()
    data class NotFound(val name: String) : FindUserResult()
    data class MalformedName(val name: String) : FindUserResult()
    // other cases that need different business-specific handling code
}

fun findUserByName(name: String): FindUserResult
```

Kotlin에서 **예외(exception)는 보통 각 호출지(call site)에서 로컬 처리가 요구되지 않는 실패들을 위해 설계되었다.** 예외는 광범위한 범위를 포함한다 — index bound 에러와 같은 논리 오류, 네트워크 등 환경 문제, 메모리 초과 오류 등.

이러한 실패들은 보통 복구할 수 없고, 로그를 남기거나 다른 방법으로 보고함으로써 중앙화된 트러블슈팅 방법으로 처리된다 — 애플리케이션을 종료하거나 애플리케이션 전체 또는 실패한 서브시스템을 재시작하는 방법이 일반적이다.

현재 실행을 중지하고 호출 수택 위로 예외를 전파하는 기본 예외 동작은 위와 같은 처리에 유용하다.

여기서 네트워크, 파일 I/O 오류와 같은 외부 환경 코너 케이스를 표현한다. 이런 에러들을 호출자에게 로컬에서 처리하도록 요구하는 것은 번거롭다. **IO 오류를 처리하는 코드로 순차적 업무 로직을 모호하게 만들어 로직을 복잡하게 만들기 때문이다.**

따라서 이를 위해 Kotlin에서 IOException과 같은 예외를 사용하는 것은 관용적(idiomatic)이다.

하지만 예외는 전역 에러 처리 코드보다 더 세분화된 수준에서 처리되는 경우가 많다. 이러한 예외들은 특정한 사용자 상호작용을 요구하거나 도메인 관련 재시도 또는 복구 코드가 필요한 경우가 많다.

> 예외가 포함하는 많은 추가 메타데이터(디버깅을 위한 스택 트레이스와 메시지)는 생성 비용이 매우 높다. 하지만 개발자가 트러블슈팅할 때 이런 메타데이터는 그 생성 비용보다 훨씬 더 유용하다.
> 
> 
> 하지만 업무 로직에서 예외를 처리하면 이 모든 메타데이터는 필요없어진다. 특정 핸들링이 필요한 실패의 경우 nullable 타입 또는 도메인 특화 클래스를 사용해라.
> 

그래서, 호출자에게 로컬 실패 처리가 요구되지 않는다면 실패는 예외로 표현되어야 한다.

```kotlin
fun findUserByName(name: String): User
```

만약 위 함수의 호출자가 여러 가지 처리를 한 후 나중에 실패를 핸들링하기를 원한다면 (첫 번째 실패에 실행을 중단하지 않고) `runCatching` 을 사용하여 실패를 잡고 Result 타입으로 캡슐화할 수 있다.

```kotlin
runCatching { findUserByName(name) }
```

## References

- [KEEP: Encapsulate successful or failed function execution
](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md)
