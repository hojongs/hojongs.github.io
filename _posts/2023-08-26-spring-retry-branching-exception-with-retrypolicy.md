---
layout: post
title: "Spring Retry: RetryPolicy로 Exception 분기 처리하기"
categories: [Spring]
tags: [Spring, Spring Retry]
---

Keyword: Spring Retry, Retryable, RetryPolicy, MaxAttemptsRetryPolicy, Backoff

Spring framework에 있는 Retry 기능을 활용하여 Elasticsearch client requset 실패 시 retry하려고 한다.

이 때 ElasticsearchStatusException의 경우 하위 type에 따라 retry 여부를 분기 처리하였다.

## Spring Retry?

<https://github.com/spring-projects/spring-retry>

GitHub README에서 공식 가이드를 확인할 수 있다. Spring app을 위한 retry 기능을 제공한다.

## `@Retryable` annotation

Spring Retry를 사용하는 가장 간단한 방법은 `@Retryable` 어노테이션을 사용하는 것이다. annotation attribute를 통해 maxAttempts 등을 설정할 수 있다.

```kotlin
@EnableRetry
@SpringBootApplication
class App
```

```kotlin
@Retryable(maxAttempts = 2, include = [IOException::class, ElasticsearchStatusException::class]
fun query() { ... }
```

## RetryPolicy

Exception type 수준의 retry 여부 분기 처리는 `@Retryable` 어노테이션의 attribute 설정만으로도 가능하다. 하지만 좀더 복잡한 로직 구현이 필요하다면 RetryPolicy를 알아야 한다.

메소드 호출 실패 후 재시도 여부는 RetryPolicy 인터페이스를 통해 구현된다. RetryPolicy.canRetry() 메소드의 return 값에 따라 결정된다.

즉, 재시도 가능 여부의 로직을 컨트롤하고 싶으면 RetryPolicy를 변경하거나 구현하면 된다.

RetryPolicy의 기본 구현체는 다양하며, SimpleRetryPolicy, MaxAttemptsRetryPolicy 등이 있다.

Exception 세부 분기 처리 로직을 추가할 것이므로 이번 포스트에서는 MaxAttemptsRetryPolicy를 상속받아서 구현하였다.

## RetryTemplate

`@Retryable` 어노테이션에는 RetryPolicy를 지정하기 위한 attribute가 없다. 이 경우 어노테이션 대신 RetryTemplate을 직접 사용하는 것이 더 편리하다.

RetryTemplate은 bean으로 등록되어 있지 않다. 직접 instance를 생성해서 사용하면 된다.

RetryTemplate에 RetryPolicy 설정 방법:

```kotlin
private val retryTemplate = RetryTemplate().apply {
    setRetryPolicy(ElasticsearchRetryPolicy())
}
```

## RetryPolicy 구현

원하는 retry 로직은 아래와 같다.

- 최대 4회 재시도 한다.
- AND 발생한 exception이 ElasticsearchStatusException type일 경우, exception이 하위 type을 참조하여 분기처리할 것이다. (index_closed_exception이 있고, circuit open 등 다양한 type이 있다.)
  - exception이 IOException type일 경우 재시도한다.
  - 이 외에 type일 경우 재시도하지 않는다.

구현하기 위해 알아야 하는 정보는 아래와 같다.

- 횟수 기준 재시도: MaxAttemptsRetryPolicy 클래스를 상속받아 재사용하면 된다.
- 파라미터로 들어오는 RetryContext의 RetryContext.getLastThrowable()를 통해 exception instance를 참조할 수 있다.
- canRetry 메소드는 최초 호출 전에도 호출된다. 이 경우 getLastThrowable()은 null을 return한다. (즉, null일 경우 true를 return 해야 한다.)
  - RetryContext.getLastThrowable() 주석 참고.

```java
/**
  * Accessor for the exception object that caused the current retry.
  * @return the last exception that caused a retry, or possibly null. It will be null
  * if this is the first attempt, but also if the enclosing policy decides not to
  * provide it (e.g. because of concerns about memory usage).
  */
Throwable getLastThrowable();
```

이를 Kotlin으로 구현한 예제는 아래와 같다. 참고로 ES client는 `RestHighLevelClient` 클래스를 사용했다. (search 메소드를 호출함)

```kotlin
class ElasticsearchRetryPolicy : MaxAttemptsRetryPolicy(4) {
    override fun canRetry(context: RetryContext): Boolean {
        return super.canRetry(context) && when (val t = context.lastThrowable) {
            is ElasticsearchStatusException -> {
                // ...
            }

            else -> t == null || t is IOException
        }
    }
}
```

위에서 언급했듯이, RetryPolicy를 RetryTemplate에 적용한다.

```kotlin
private val retryTemplate = RetryTemplate().apply {
    setRetryPolicy(ElasticsearchRetryPolicy())
}
```

RetryTemplate를 사용해 query 메소드를 호출한다.

```kotlin
val searchHits = retryTemplate.execute(
    RetryCallback { logSearchService.query() }
)
```

참고로 RetryTemplate는 기본적으로 RetryPolicy로 SimpleRetryPolicy(3), NoBackOffPolicy()를 사용한다.

NoBackOffPolicy는 retry 시 backoff를 하지 않고 delay 없이 즉시 retry한다. 

> backoff는 retry 요청을 받는 서버의 급격한 과부하를 방지하기 위해 적용되는 테크닉이다.

## 추가 작업: RetryListener, BackoffPolicy

실제로 retry가 잘 되고 있는지 확인하기 위해 로그를 추가할 수도 있다. 또한 즉시 retry 시 서버에 과부하가 발생하거나 동일한 에러로 retry가 실패할 수도 있다. 이를 방지하기 위해 backoff를 설정할 수도 있다.

```kotlin
private val retryTemplate = RetryTemplate().apply {
    setRetryPolicy(ElasticsearchRetryPolicy())
    setListeners(arrayOf(LoggingRetryListener()))
    // backOff for index_closed_exception
    setBackOffPolicy(FixedBackOffPolicy().apply { backOffPeriod = backOff /* 5000L */ })
}
```

```kotlin
class LoggingRetryListener : RetryListenerSupport() {
    override fun <T : Any?, E : Throwable> onError(
        context: RetryContext,
        callback: RetryCallback<T, E>,
        throwable: Throwable,
    ) {
        // ...
    }
}
```

