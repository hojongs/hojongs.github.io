---
layout: post
title: "[TIL] Logback Automatic Configuration Initialization"
categories: [Logback]
tags: [TIL, Logback]
---

배경:

팀원의 코드를 리뷰하던 중, Unit test에서 `resources/logback-test.xml` 파일을 생성하고 이를 활용하여 테스트하는 것을 발견

Spring app이 아닌데 어떻게 logback configuration을 인식한 것일까?

<https://logback.qos.ch/manual/configuration.html#auto_configuration>
