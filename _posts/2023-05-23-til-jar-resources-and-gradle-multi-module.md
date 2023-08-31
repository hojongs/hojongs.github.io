---
layout: post
title: "[TIL] Gradle: multi-module 프로젝트 Jar의 resources 동작"
categories: [Gradle]
tags: [TIL, Gradle, Jar, Multi-module]
---

multi-module 프로젝트에서 bootJar를 빌드하는 모듈이 있고, 이 모듈이 의존하는 다른 모듈이 있다

bootJar 모듈은 resources 디렉토리에 application.properties, logback.xml과 같은 파일들이 있다.

이 때, 의존 모듈에 있는 resources 디렉토리에 있는 파일들은 bootJar에 포함될까?

## Multi-module 구조



https://docs.oracle.com/javase/tutorial/deployment/jar/unpack.html
https://www.baeldung.com/docker-layers-spring-boot
