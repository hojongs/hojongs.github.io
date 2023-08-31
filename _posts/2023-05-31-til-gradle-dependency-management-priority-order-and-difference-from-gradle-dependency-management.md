---
layout: post
title: "[TIL] Spring: dependency-management 선언과 적용 우선순위, vs Gradle dependency management 차이점"
categories: [Spring]
tags: [TIL, Spring, Gradle, Dependency Management]
---

Spring boot 서버를 개발하고 있다면 [Spring dependency management plugin]에 대해 들어봤을 것이다. 이 플러그인을 통해 의존성들의 버전을 통합 관리할 수 있다.

bom을 import하여 의존성들이 bom에 명시된 버전을 사용하도록 할 수 있다. 이 때, 여러 개의 bom에서 동일한 의존성의 여러 개의 버전을 선언하고 있다면 어떤 bom에 있는 버전으로 resolve 될까? 즉, 이것은 bom 선언 우선순위에 의해 결정된다.

한편, Gradle은 4.6부터 자체 dependency management 기능을 제공하고 있다. Spring에서 제공하는 위 플러그인과 차이는 무엇일까?

이 포스트에서는 아래 두 개 주제에 대해서 정리하였다.

- Spring dependency management에 선언된 bom들의 우선순위
- [Spring dependency management plugin] vs [Gradle built-in dependency management]

## Spring dependency management: BOM 선언과 적용의 우선순위

관련 내용을 공식 문서의 아래 페이지에서 찾을 수 있다

<https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/#dependency-management-configuration-bom-import-multiple>

> If you import more than one bom, the order in which the boms are imported can be important. The boms are processed in the order in which they are imported. If multiple boms provide dependency management for the same dependency, the dependency management from the last bom will be used.

나중에 선언된 bom이 이전 bom을 override 방식이다. 

### Maven properties

위 내용의 하단에 `4.2.2. Overriding Versions in a Bom`에 대한 내용이 언급된다. 관심있는 내용이라서 함께 찾아본 관련 링크들만 첨부해놓는다. (포스트의 주제를 벗어나므로)

- <https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/#dependency-management-configuration-bom-import-override>
- <https://maven.apache.org/pom.html#Properties>
- <https://docs.gradle.org/current/userguide/publishing_maven.html>
- <https://docs.gradle.org/current/userguide/migrating_from_maven.html#migmvn:profiles_and_properties>
- <https://cornswrold.tistory.com/228>

## [Spring dependency management plugin] vs [Gradle built-in dependency management]

동일한 질문이 플러그인 저장소에 이미 올라와있었다. 간단히 말하면 아직은 Gradle에서 제공하지 않는 기능 (Maven exclusion)이 있기 때문에 여전히 대안으로 존재한다고 한다.

<https://github.com/spring-gradle-plugins/dependency-management-plugin/issues/211>

### resolution strategy 차이점

resolutionStrategy 또한 plugin의 것과 Gradle의 것이 각각 존재한다. 이 둘의 차이점은 뭘까? 아래 문서에서 힌트를 얻을 수 있다.

<https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/#dependency-management-configuration-import-bom-resolution-strategy>

> The plugin uses separate, detached configurations for its internal dependency resolution. You can configure the resolution strategy for these configurations using a closure. If you’re using a snapshot, you may want to disable the caching of an imported bom by configuring Gradle to cache changing modules for zero seconds, as shown in the following example:

서로 다른 레이어에서 의존성 버전 설젇을 관리하기 때문에, 각 resolutionStrategy은 동작에 차이가 있다. 이로 인해 동일한 bom을 사용하더라도 최종적으로 resolution 되는 버전이 다르다.

[Spring dependency management plugin]: https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/

[Gradle built-in dependency management]: https://docs.gradle.org/current/userguide/core_dependency_management.html
