---
layout: post
title: "Spring Boot: Overriding Dependency Versions with Gradle Properties"
categories: [Spring Boot]
tags: [Spring Boot, Dependency Management, Gradle]
---

의존성 관리는 멋지지 않지만 굉장히 중요하고 컨트롤하기 어려운 문제 중 하나다.

특히 수십 개의 독립적인 모듈로 이루어진, 그리고 수백 개의 의존성을 가진 Spring boot 의존성들은 의존성이 매우 복잡하다.

Spring boot를 사용하는 서버의 의존성도 이 복잡한 의존성 그래프를 피해가기 어려운데, 이를 해결하기 위해 Spring boot에서는 몇 가지 기능들을 제공한다.

본 포스트에서는 그 중에서도 `customizing(overriding) managed dependency version`에 대해 알아본다.

## Customizing Managed Versions

<https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/#managing-dependencies.dependency-management-plugin.customizing>

Spring boot는 수많은 의존성들의 호환성을 보장해주기 위해 이 의존성들의 버전 지정 묶음을 제공한다. `spring-boot-dependencies`라는 모듈에서 이를 제공한다.

하지만 종종 우리는 일부 의존성의 버전을 변경하고 싶은 니즈가 발생한다. 해당 버전에 취약점이 발견되었거나, 해당 버전에 버그가 발견되었을 때 특히 그렇다.

또는 Kotlin 새로운 버전이 릴리즈 되었을 때 Kotlin 버전업을 시도해볼 수 있다.

예를 들면, Spring boot 2.4.13에서는 Kotlin coroutine 1.4.3 버전을 사용한다. 

<https://docs.spring.io/spring-boot/docs/2.4.13/reference/htmlsingle/#dependency-versions>

하지만 이 버전에는 `awaitSingleOrNull` 함수 관련 버그가 있어서, Compile-time에 NPE를 잡지 못하고 Runtime에서 NPE를 마주하게 된다.

<https://medium.com/jongho-developer/kotlin-mono-awaitsingleornull-%EC%9D%98-non-nullable-%EB%B0%98%ED%99%98-%EB%AC%B8%EC%A0%9C-92b172cf3890>

따라서 coroutine 의존성 버전을 올려주면 좋은데, Gradle dependencies 블록에 `implementation(org.jetbrains.kotlinx:kotlinx-coroutines-core:1.5.0)`을 추가하는 방법도 가능하다. 하지만 이 방법엔 문제가 있다.

- coroutine을 포함한 다양한 의존성들은 의존성이 패키지처럼 되어 있어서 여러 의존성들의 버전이 함께 관리되어야 한다. 즉, 모든 `kotlinx-coroutines-...` 의존성들의 버전을 선언해야한다. -> Gradle build script가 과도하게 길어진다. 이를 지키지 않으면, 호환성 에러가 발생할 수 있다.
- 종종 (다른 의존성이 요구하는 동일 의존성의 버전에 의해) 의존성 버전 conflict가 발생하여 의도하지 않은 버전으로 의존성 버전이 resolve되기도 한다. 1.5.0을 사용하도록 선언했으나 여전히 1.4.3을 사용하는 것이다.
- Kotlin 언어 버전을 올릴 때는 Kotlin plugin script config에서 사용하는 Kotlin 버전만 낮게 resolve되어 Gradle Kotlin DSL 사용에 어려움을 겪을 수도 있다.

이러한 문제를 해결하기 위해, Spring boot에서는 아래와 같은 property들을 제공한다.

`kotlin.version`
`kotlin-coroutines.version`

이 값을 설정하여 단 한 줄로 관련된 모든 의존성의 버전을 컨트롤할 수 있다.

## Version properties

어떻게 구현되어있는지 궁금해서 조금 코드를 읽어보았다.

<https://github.com/spring-projects/spring-boot/blob/9643dbeed26268f37d37c22385704de8911b8f21/spring-boot-project/spring-boot-dependencies/build.gradle>

spring-boot-dependencies 모듈에서 bom extension을 통해 library들을 정의한다.

coroutine의 경우 coroutine bom을 import하고 있다. kafka의 경우 10개가 넘는 kafka 관련 모듈들을 그룹으로 묶고 있다.

각각을 library라는 것으로 정의하고 있는데, 관련 코드를 따라가보았다.

Gradle 상단에서 spring boot bom 플러그인을 선언해서 사용하고 있다. 이는 buildSrc에 정의된 플러그인이다.

spring boot bom plugin: <https://github.com/spring-projects/spring-boot/tree/9643dbeed26268f37d37c22385704de8911b8f21/buildSrc/src/main/java/org/springframework/boot/build/bom>

이 중 Library라는 파일(클래스)를 보면 된다: <https://github.com/spring-projects/spring-boot/blob/9643dbeed26268f37d37c22385704de8911b8f21/buildSrc/src/main/java/org/springframework/boot/build/bom/Library.java>

아래와 같은 과정을 통해 property 이름이 생성됨을 확인할 수 있었다.

```java
		this.versionProperty = "Spring Boot".equals(name) ? null
				: name.toLowerCase(Locale.ENGLISH).replace(' ', '-') + ".version";
```

`spring-boot-dependencies` 모듈은 build.gradle 파일 하나로 이루어진 단순한 모듈이고, 로직은 모두 buildSrc 이하에 있다. (결국 bom plugin에 있을 것이다.)

코드를 다 읽어보지는 않았지만, `spring-boot-dependencies`의 기능을 쉽게 생각하면 아래와 같다.

- version property들을 불러온다. (없으면 기본값 사용)
- managed dependency들의 버전을 해당 version으로 지정한다.

## Conclusion

Spring boot의 의존성을 쉽게 관리하기 위해 알아야 할 기능이기도 하지만, Spring boot처럼 사내에서 연관된 모듈들을 여러 개 관리할 때에도 유용한 아이디어다.

특히 여러 의존성들이 공통 의존성을 가질 때, 공통 의존성의 버전 충돌 문제를 해결하는 데에 유용할 수 있다.

각각의 의존성에는 내부 의존성들의 version이나 scope을 어떻게 선언해야 할지, 예를 들면 compileOnly로 선언해야할지 implementation으로 선언해야할지, 저장소는 모노레포로 구성해야할지 등의 여러 가지 결정해야할 설계점들이 있다.

Spring boot 저장소를 벤치마킹하여 사내 모듈들의 의존성 관리에 응용해보기 위해, `spring-boot-dependencies` 모듈의 의존성 관리 기술에 대하여 알아보았다.

우선 한 가지 insight는, 서로 의존하는 모듈들은 하나의 저장소에서 하나의 브랜치(히스토리 로그)로 관리하는 것이 적합해보인다. 즉 공통 코드를 공유하는 여러 모듈들이 있다면, 그 모듈들은 하나의 저장소에서 관리하는 것이 가장 쉬울 것이다.

이후의 생각에 대해서는 관련 작업을 진행하면서, 별도 포스트로 작성해볼 예정이다.
