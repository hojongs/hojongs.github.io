---
layout: post
title: "[TIL] Gradle: Java(Kotlin) targetCompatibility/sourceCompatibility"
categories: [Gradle]
tags: [TIL, Gradle, Java, Kotlin]
---

        targetCompatibility = JavaVersion.VERSION_1_8

A->B로 의존할 때
A의 targetCompatibility가 1.8일 때
B의 targetCompatibility가 11이면 컴파일 불가능

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
