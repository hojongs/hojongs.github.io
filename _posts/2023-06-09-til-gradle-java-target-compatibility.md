---
layout: post
title: "[TIL] Gradle: Java(Kotlin) targetCompatibility/sourceCompatibility"
categories: [Gradle]
tags: [TIL, Gradle, Java, Kotlin]
---

Java 또는 Kotlin으로 구성된 Gradle 프로젝트에서 아래와 같은 빌드 스크립트를 본 적이 있을 것이다.
이 중 targetCompatibility에 대해서 알아볼 것이다. 이 프로젝트의 jar 파일을 실행하는 JVM runtime version 또는 다른 jar artifact를 의존할 때 이 targetCompatibility가 중요하다.

이 설정은 Kotlin/JVM을 개발할 때도 중요하다.

```kotlin
// build.gradle.kts
tasks {
    withType<JavaCompile> {
        sourceCompatibility = "11"
        targetCompatibility = "11"
    }

    withType<KotlinCompile> {
        kotlinOptions {
            jvmTarget = "11"
        }
    }
}
```

server, lib 두 개의 Gradle 프로젝트 모듈 또는 jar artifact가 있다고 하자.

server가 lib 모듈에 의존할 때

- server의 targetCompatibility가 1.8일 때
- lib의 targetCompatibility가 11이면 server는 컴파일 불가능하다.

아래와 같은 에러를 만나게 된다.

```
Execution failed for task ':compileKotlin'.
> Could not resolve all files for configuration ':compileClasspath'.
   > Could not resolve {MODULE}.
     Required by:
         project :
      > No matching variant of {MODULE} was found. The consumer was configured to find an API of a library compatible with Java 8, preferably in the form of class files, and its dependencies declared externally, as well as attribute 'org.jetbrains.kotlin.platform.type' with value 'jvm' but:
          - Variant 'apiElements' capability {MODULE} declares an API of a library, packaged as a jar, and its dependencies declared externally, as well as attribute 'org.jetbrains.kotlin.platform.type' with value 'jvm':
              - Incompatible because this component declares a component compatible with Java 11 and the consumer needed a component compatible with Java 8
          - Variant 'runtimeElements' capability {MODULE} declares a runtime of a library, packaged as a jar, and its dependencies declared externally, as well as attribute 'org.jetbrains.kotlin.platform.type' with value 'jvm':
              - Incompatible because this component declares a component compatible with Java 11 and the consumer needed a component compatible with Java 8
```

---

KotlinCompile type task들의 kotlinOptions.jvmTarget에 대해서도 비슷한 이슈가 있다.

Kotlin의 jvmTarget을 `"11"`로 설정하면 Java 8 runtime에서 실행할 수 없다.
Apache Flink 구 버전이 Java 8에서 실행되고 있어서 위 에러를 만났었다.

<https://flink.apache.org/>
