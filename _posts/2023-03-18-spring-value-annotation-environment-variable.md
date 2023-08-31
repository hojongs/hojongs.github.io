---
layout: post
title: "Spring: 환경변수를 @Value 어노테이션, application.properties에서 주입받기"
categories: [Spring]
tags: [Spring, Kotlin]
---

## Access environment variables within Spring Boot

Spring에서 개발할 때, 환경변수에 접근하는 방법은 여러 가지가 있다. `System.getenv()` 등등.

보통 환경변수는 서버 설정값을 주입하기 위해 사용한다. 서버 설정값은 application.properties에도 설정되는데, `System.getenv()`와 같은 코드로 환경변수를 주입받는다면 관리가 어려워진다.

어떤 환경변수를 주입받는지 / 환경변수를 어디서 주입받는지 / 서버를 설정하기 위해 필요한 값들은 무엇인지 알기 어려워지기 때문이다.

그래서 환경변수를 properties 파일에서 주입받은 후 코드 레벨에서는 properties 파일에서만 읽어들이는 것이 좋다.

하지만 경우에 따라 `@Value` 어노테이션에서 환경변수를 주입받는 방법을 알아두는 것도 유용할 것이다.

## 환경변수 `@Value` 어노테이션에서 주입받기 (How to inject environment variable to `@Value` annotation)

<https://stackoverflow.com/a/14617182/12956829>

```java
@Component
public class SomeClass {
    @Value("#{environment.SOME_KEY_PROPERTY}")
    private String key;

    ....
}
```

```kotlin
class SomeClass(
    @Value("#{environment.SOME_KEY_PROPERTY}")
    private key: String,
) {
    ....
}
```

### `@Value` 어노테이션 환경변수 & 기본값 (default value)

```kotlin
class SomeClass(
    @Value("\${#{environment.SOME_KEY_PROPERTY}:my-default}")
    private key: String,
) {
    ....
}
```

> Kotlin에서 `@Value` 값에 `$` 사용 시에는 `\$`로 입력해야 함을 주의

SpEL 수준에서 default value를 지정할 수도 있다.

```kotlin
@Value("#{systemProperties['unknown'] ?: 'some default'}")
private spelSomeDefault: String
```

Spring Value 어노테이션에 대한 자세한 내용은 여기를 참고: <https://www.baeldung.com/spring-value-annotation>

### 동작원리: `@Value` and SpEL

`#{ <expression string> }`의 표현식은 SpEL (Spring Expression Language)의 문법이다. `@Value` 어노테이션의 값에 SpEL을 사용할 수 있는 것이다.

`@Value`: <https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-value-annotations>

> When @Value contains a SpEL expression the value will be dynamically computed at runtime as the following example shows:

SpEL
- <https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions>
- <https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions-beandef>

## 환경변수 application.properties에서 주입받기 (How to inject environment variable to `@Value` annotation)

<https://www.baeldung.com/spring-boot-properties-env-variables>

JAVA_HOME이라는 환경변수를 주입받고 싶을 때

```properties
java.home=${JAVA_HOME}
```

properties의 property는 아래와 같이 `@Value` 어노테이션 등으로 다시 주입받으면 된다.

```kotlin
class SomeClass(
    @Value("\${java.home}")
    private javaHome: String,
) {
    ....
}
```
