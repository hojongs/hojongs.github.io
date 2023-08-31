---
layout: post
title: "[TIL] Spring: How to Inject Spring Bean into Static Variable"
categories: [Spring]
tags: [TIL, Spring]
---

Spring bean을 static variable(정적 변수)에 저장하는 것이 유용하거나 편리할 경우가 있다. 예를 들면 `Environment.activeProfiles`의 값을 그렇게 활용할 수 있다.

사실 이렇게 static variable에 의존하는 클래스는 테스트하기 더 어려워지므로 static variable이 아닌 constructor를 통해 DI 받는 것이 더 좋은 패턴이라고 할 수 있다. 하지만 간단한 케이스에서는 이 방법이 유용할 수 있다.

## Inject Spring bean in constructor

```kotlin
private val logger = KotlinLogging.logger {}

@Component
class MyUtil(environment: Environment) {
    companion object {
        private var isProd: Boolean? = null
        fun isProd(): Boolean = isProd
            ?: (System.getProperty("spring.profiles.active", "prod") == "prod")
    }

    init {
        isProd = environment.activeProfiles.contains("prod")
        logger.info("isProd: $isProd")
    }
}
```

가장 간단한 방법은 위와 같이 static variable을 가진 클래스를 bean으로 등록하고 생성자에서 static variable을 초기화해주는 것이다.

## Note: Bean registration order

이렇게 초기화된 static variable을 사용할 때 주의할 점이 있다. bean 등록(초기화) 순서이다.

```kotlin
@Service
class MyService {
    private val isProd = MyUtil.isProd() // static variable 초기화 전에 호출될 수도 있음
    
    fun f() {
        println(isProd)
    }
}
```

MyUtil, MyService bean들의 초기화 순서가 정의되어 있지 않기 때문에 초기화 전에 참조가 발생할 수 있다. Spring bean 클래스에 `@Order` 어노테이션은 초기화 순서가 아닌 정렬 순서를 정의한다.

`@Order in Spring`: <https://www.baeldung.com/spring-order>

### 방법 1: `@DependsOn`

`@DependsOn`이라는 어노테이션을 통해 bean 생성 순서를 정의할 수는 있다.

`Controlling Bean Creation Order with @DependsOn Annotation`: <https://www.baeldung.com/spring-depends-on>

### 방법 2: Do not refer while bean initialization (초기화 중에 참조하지 않기)

또다른 방법으로, 가능하다면 해당 참조를 초기화 이후 시점으로 연기하는 것이다.

```kotlin
@Service
class MyService {
    fun f() {
        println(MyUtil.isProd())
    }
}
```

여전히 변수로 사용하고 싶다면, `lazy delegator`를 활용할 수도 있다.

lazy prop: <https://kotlinlang.org/docs/delegated-properties.html#observable-properties>

```kotlin
@Service
class MyService {
    private val isProd: Boolean by lazy { MyUtil.isProd() }

    fun f() {
        println(isProd)
    }
}
```

## Conclusion

Spring bean을 static variable로 DI하고, 이렇게 사용할 때의 유의사항들과 노하우들을 알아보았다.

이렇게 사용하는 방식은 클래스의 테스트를 더 어렵게 만들 수 있기 때문에, 꼭 필요한 상황에만 사용하는 것이 좋다.
