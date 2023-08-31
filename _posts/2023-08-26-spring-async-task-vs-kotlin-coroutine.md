---
layout: post
title: "Spring Async Task vs Kotlin coroutines"
categories: [Spring]
tags: [Spring, Kotlin, Async Task, Kotlin, Coroutine, Elasticsearch, Concurrent Processing, Asynchronous]
---

Keyword: Spring Framework, Async Task, Kotlin, Coroutine

특정 서비스에서 **여러 개의 Elasticsearch query**를 요청하는 API가 있다. 이 API는 query를 순차적으로 요청하고 있었고, query의 개수가 증가하면서 latency가 점점 증가하는 문제가 있었다. 쿼리 성능을 높이기 위해 Elasticsearch 쿼리를 병렬로 수행하도록 변경하였다. **이를 통해 15초~100초 정도 걸리던 API를 p95 10s, p99 12s 수준으로 개선했다.**

병렬 수행을 구현할 때 처음에는 Spring Async Task 기능을 사용했다가 **Kotlin coroutine**으로 변경했다. 이 과정에서 알아본 두 기능의 장단점 비교를 정리했다.

## Spring Async Task

Spring은 비동기적으로 Task를 실행할 수 있는 기능을 지원한다. `@Async` 어노테이션을 통해 특정 메소드가 다른 클래스에서 호출될 때 비동기적으로 동작하도록 선언 할 수 있다.

<https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async>

이러한 Task들은 TaskExecutor에 의해 실행된다. TaskExecutor는 Task들을 별도의 스레드에서 실행할 수 있다.

## TaskExecutor

<https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-task-executor>

### Thread pool config: ThreadPoolTaskExecutor

OOM 등을 방지하기 위해 Task들이 실행될 thread들을 관리할 필요가 있다. Task 실행을 위해 너무 많은 스레드가 동시에 생성되면 OOM error가 발생할 수도 있고, 너무 많은 스레드는 애플리케이션 리소스를 비효율적으로 사용하여 성능 저하가 발생할 수 있다. (context switching)

TaskExecutor 구현체 중 thread pool을 관리하는 ThreadPoolTaskExecutor를 활용할 수 있다.

task들이 특정 개수의 thread pool에서만 실행되도록 하려면 아래와 같이 설정해야 한다.

> `@Async` 어노테이션이 동작하려면 `@EnableAsync` 어노테이션이 설정되어야 한다.

```kotlin
@Configuration
@EnableAsync
class AsyncTaskConfig {
    @Bean
    fun taskExecutor(): TaskExecutor {
        val threadCount = 4
        return ThreadPoolTaskExecutor().apply {
            setThreadNamePrefix("async-")
            corePoolSize = threadCount
            maxPoolSize = threadCount
        }
    }
}
```

호출은 아래와 같다

```kotlin
@Async
fun f() { ... }


fun caller() {
    repeat(5) {
        service.f()
    }
}
```

### Await all async tasks

caller(호출자)에서 이러한 비동기 호출들이 완료되기를 기다리려면, @Async 어노테이션을 설정한 메소드의 return type을 Future type으로 설정해주면 된다. (Future.get() 메소드가 await 기능을 한다)

### `@Async` annotation & Proxy call: 클래스 분리

또한 중요한 점은, `@Async` 어노테이션은 다른 Spring 어노테이션들이 그러하듯, CGLib proxy를 통해 구현된다. 즉, 클래스 내부 호출에서는 동작하지 않는다. (`@Transactional` 어노테이션을 사용해본 사람들은 한 번쯤 경험해봤을 것이다.)

따라서, 해당 메소드는 별도의 클래스로 분리되어야 한다.

혹은 TaskExecutor bean을 직접 DI받아서 호출하는 방법이 있지만, `@Async`에 비해 비교적 번거롭다.

### Graceful shutdown for Async task

그리고 Async task가 graceful shutdown을 지원하려면 아래와 같이 설정해줘야 한다. TaskExecutor에 추가 설정이 필요하다.

```properties
server.shutdown=graceful
```

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor

@Bean
fun taskExecutor(): ThreadPoolTaskExecutor {
    val executor = ThreadPoolTaskExecutor()
    // 기타 구성 설정. 예를 들어:
    executor.corePoolSize = 5
    executor.maxPoolSize = 10
    executor.setQueueCapacity(25)
    executor.setThreadNamePrefix("MyAsync-")

    // 작업 완료 대기 활성화
    executor.setWaitForTasksToCompleteOnShutdown(true)

    // 작업 완료 대기 시간 설정 (예: 20초)
    executor.setAwaitTerminationSeconds(20)

    return executor
}
```

### Thread context (MDC) propagation

Thread가 변경되면 thread local에 의존하는 context들은 유실된다. 대표적인 것이 MDC와 Tracing context이다.

```kotlin
executor.setTaskDecorator(MDCTaskDecorator())
```

MDC 전파를 위한 TaskDecorator는 존재하지 않으므로, 직접 구현해야 한다. 아래는 예시이다.

```java
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            if (contextMap != null) {
                MDC.setContextMap(contextMap);
            }
            try {
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

### 중간 요약

Spring async task를 쓰려면 다양한 부분을 고려해야 한다. 요약하면 아래와 같다. 5가지나 되므로 호출 시점 기준으로 분류해보았다.

- 비동기 호출 전 (config)
  - Thread pool config: ThreadPoolTaskExecutor
  - `@Async` 어노테이션 proxy call: 클래스 분리
- 비동기 호출 중
  - Thread context (e.g. MDC) propagation
- 비동기 호출 후
  - Await async task: Future
  - Graceful shutdown

생각보다 많다. 더 있을 수도 있다.

이 측면에서, Kotlin coroutine과 비교해보았다. 결론부터 말하면 Kotlin coroutine이 위 부분들을 고려하며 코딩하기 편했다.

## Kotlin coroutine

coroutine은 thread가 아닌 light-weight thread에서 실행된다고 보통 설명한다.

하지만 Thread blocking call이 존재할 경우에는 coroutine을 Thread pool에서 실행하도록 설정하여 실행해야 한다.

### Dependency

kotlin coroutine을 사용하기 위해 Gradle build script에 의존성부터 추가해주자.

의존성 버전 Spring Boot에 의해 관리되므로 버전 지정이 불필요하다.

```kotlin
    // implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
```

### Thread pool config: Dispatcher

Coroutine에서 Spring의 TaskExecutor와 대응되는 객체는 Dispatcher이다. 기본적으로 Dispatchers.IO를 제공해주며, Dispatchers.IO는 CPU core * 2개의 thread pool로 구성되지만 최소 64개의 스레드를 가진다.

<https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html>

newFixedThreadPoolContext() 메소드를 통해 스레드 풀을 만들 수 있고, thread name prefix를 지정할 수 있다.

```kotlin
@OptIn(ObsoleteCoroutinesApi::class)
private val ioDispatcher = newFixedThreadPoolContext(threadCnt, "io")
```

## Call directly within classes

Coroutine은 `@Async` 어노테이션 대신, coroutine builder 중 launch 혹은 async를 사용하면 된다. (여기서는 메소드를 suspend function으로 변경해줄 필요없다.)

> launch는 fire-and-forget, async는 코루틴의 return value를 사용해야 할 경우 사용한다.

```kotlin
fun caller() {
    runBlocking {
        launch(Dispatchers.IO) { f(i) }
    }
}

fun f(i: Int): String {
    return "$i"
}
```

Spring `@Async` 어노테이션은 proxy를 통해 구현되므로 caller는 클래스 외부에 있어야 했다. 하지만 Kotlin coroutine은 그렇지 않다. 클래스 내에서 호출하더라도 비동기로 호출할 수 있다.

> 여담: runBlocking(Dispatchers.IO) vs launch(Dispatchers.IO): runBlocking block 자체가 어느 스레드에서 실행될 지를 결정한다. 위 시나리오에서 runBlocking 블록 자체는 별도 스레드에서 실행될 필요가 없다. 이 차이로 인해 병렬도가 1 차이날 수 있다.

### Coroutine: structured concurrency

coroutine을 사용할 경우, structured concurrency 개념에 대한 이해가 필요하다. 그래야 의도하지 않은 코루틴 완료 대기 또는 코루틴 cancel 문제를 피할 수 있다.

Coroutine은 실행 중 cancel 될 수 있고 structured concurrency를 지원한다. coroutine이 cancel 될 경우 CancellationException 예외가 발생하지만 coroutine이 thread를 block할 경우 이러한 cancellation 신호를 수신할 수 없음을 유의해야 한다.

또한 coroutine이 structured concurrency를 지원하기 때문에, coroutine을 launch로 실행하거나 async로 실행 후 await 하지 않아도 runBlocking 함수는 모든 코루틴이 완료되기를 기다린다.

<https://kotlinlang.org/docs/coroutines-basics.html#structured-concurrency>

Structured concurrency는 훌륭한 기능이지만, 코루틴 동작을 이해하기 위해 Structured concurrency에 대한 이해도 필요하다.

## Thread context (e.g. MDC) propagation

kotlin-coroutine-slf4j 의존성에서 MDCContext라는 클래스를 제공한다.

아래와 같이 쉽게 사용할 수 있다.

```kotlin
fun caller() {
    runBlocking {
        launch(Dispatchers.IO + MDCContext()) { f(i) }
    }
}
```

coroutine이 suspend/resume 될 때마다 스레드에 MDC를 설정해준다.

## Await async task

위에서 언급했듯 Coroutine은 structured concurrency를 지원한다. 따라서 coroutine scope (runBlocking)은 종료하기 전에 모든 코루틴이 완료될 때까지 기다린다. 별도로 await을 호출할 필요 없다.

```kotlin
fun caller() {
    runBlocking {
        (0..5).map { i ->
            launch(ioDispatcher + MDCContext()) { f(i) }
        }
    }
    // 모든 코루틴이 완료된 후 도달한다
}
```

## Graceful shutdown

Spring app은 아래 property를 통해 graceful shutdown을 활성화할 수 있다.

```
# application.properties 파일의 경우
server.shutdown=graceful
```

하지만 Spring에서 Kotlin coroutine scope들을 gracefully shutdown히도록 지원해주는 기능은 없다. 이는 별도의 구현이 필요하다.

## 요약

- Spring Async Task: 
  - Thread pool config: ThreadPoolTaskExecutor bean 등록 및 사용
  - 비동기 호출 방법: `@Async` 사용 및 클래스 분리, 혹은 TaskExecutor 직접 호출
  - Thread context (e.g. MDC) propagation: TaskDecorator 직접 구현 및 설정 필요
  - Await async task: Future return하도록 변경 및 await 호출
  - Graceful shutdown: Spring app 및 TaskExecutor에 Graceful shutdown 활성화 설정
- Kotlin coroutines:
  - Thread pool config: Dispatchers.IO 사용 또는 Dispatcher 설정 및 사용
  - 비동기 호출 방법: runBlocking, launch/async 사용하여 호출
  - Thread context (e.g. MDC) propagation: MDCContext를 사용하여 
  - Await async task: return value가 필요하면 await() 또는 awaitAll() 메소드를 사용하자. (suspending function은 그저 일반 메소드처럼 호출하면 된다.) 그렇지 않다면 structured concurrency로 인해 coroutine scope은 종료되기 전에 모든 coroutine의 완료를 기다린다.
  - Graceful shutdown: 별도로 구현해야 한다.
- 비교 의견 (주관적)
  - Thread pool config: coroutine 측 설정이 더 간단함
  - 비동기 호출 방법: coroutine 측 설정이 더 간단함
  - Thread context (e.g. MDC) propagation: coroutine 측 설정이 더 간단함 (MDCContext)
  - Await async task: coroutine 측 설정이 더 간단함 (Future로 return type 변경 및 await 호출해야 하므로)
    - Coroutine을 사용한다면 Structured concurrency 동작에 대해 이해할 필요가 있다.
  - Graceful shutdown: Spring Async Task 측 설정이 더 간단함. coroutine은 직접 구현해야 함.
  - **종합: 코드 작성 측면에서는 coroutine이 더 간단하다. 하지만 Graceful shutdown 설정은 Spring Async Task가 더 간단하다.**
