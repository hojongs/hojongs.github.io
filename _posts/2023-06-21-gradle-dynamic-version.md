---
layout: post
title: "[TIL] Gradle: Understanding dependency resolution - dynamic version"
categories: [Gradle]
tags: [Gradle, Dependency Management]
---

Gradle의 의존성 resolution에 대해서 쓸 내용이 아주 많은데, 이번 포스트에서는 간단하게 dynamic version에 대해서만 다뤄본다.

<https://docs.gradle.org/current/userguide/dynamic_versions.html>

dynamic version은 `2.+`와 같이 범위로 지정될 수도 있고 `latest.integration`과 같이 최신 버전을 resolve하도록 지정될 수도 있다.

> 위 링크에 `changing version`이라고 언급되어있는 것은 쉽게 `SNAPSHOT` 모듈(라이브러리)라고 생각하면 된다.

Dynamic version을 사용할 때 아래 유의사항을 명심해야 한다. dynamic version 사용 시 빌드에서 사용될/사용된 의존성의 버전을 예측하기 어려워진다.

> Using dynamic versions and changing modules can lead to unreproducible builds. As new versions of a particular module are published, its API may become incompatible with your source code. Use this feature with caution!

## Dynamic version cache

기본적으로 dynamic version의 resolution은 24시간 캐시된다는 점도 알아둘 필요가 있다. 이 캐시가 남아있는한 새로운 버전을 publish해도 그 버전이 resolve되지 않기 때문이다.

> By default, Gradle caches dynamic versions of dependencies for 24 hours. Within this time frame, Gradle does not try to resolve newer versions from the declared repositories. The threshold can be configured as needed for example if you want to resolve new versions earlier.

물론 아래와 같은 방법으로 cache TTL을 변경할 수는 있다.

```kotlin
configurations.all {
    resolutionStrategy.cacheDynamicVersionsFor(10, "minutes")
}
```

---

<https://docs.gradle.org/current/userguide/dependency_resolution.html#sec:how-gradle-downloads-deps>

Gradle은 dynamic version resolution을 위해, 선언된 모든 repository들로부터(e.g. mavenLocal, mavenCentral, ...) 의존성의 metadata들을 다운로드한다.

그리고 이 버전 정보들 (metadata들)은 캐시된다. 이 메타데이터들이 저장되는 위치는 아마도 아래 path인 듯하다.

```
$HOME/.gradle/caches/modules-2/metadata-{version}/descriptors/{package}/{dependency}/{version}
```

[[TIL] Gradle: dependency cache path in Home directory](/posts/til-gradle-dependency-cache-path-in-home-directory/) 에서 소개했던 `modules-2` directory 이하에 있다.

Dependency cache: <https://docs.gradle.org/current/userguide/dependency_resolution.html#sec:dependency_cache>

Gradle module metadata: <https://docs.gradle.org/current/userguide/publishing_gradle_module_metadata.html>

<https://docs.gradle.org/current/userguide/dependency_resolution.html#sub:ephemeral-ci-cache>

> Note that Gradle caches the version information, more information can be found in the section Controlling dynamic version caching.

## component selection rule

`1.+` 버전 중 특정 버전 (e.g. `1.5`)은 reject 해야할 때와 같이 특수한 케이스에 대해 사용 가능하다고 한다.

## Conclusion

Gradle에서 dependency의 version을 dynamic version으로 선언했을 때 일어나는 동작들(metadata cache, version resolution)과 관련 기능들, 사용 시 유의사항에 대해서 알아보았다.
