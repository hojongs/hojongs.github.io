---
layout: post
title: "Spring Cloud: Bootstrap이란? (bootstrap properties vs application properties)"
categories: [Server development]
tags: [Spring Cloud, Spring Boot]
---

## Bootstrap properties vs application properties

### Application properties

<https://www.baeldung.com/spring-cloud-bootstrap-properties>

Spring Boot에서 Application properties 파일은 Spring application context를 설정하기 위한 설정 파일이다.

Spring boot application이 실행될 때 설정값들이 전달되는데, application properties 파일은 이 설정값의 source 중 하나이다.

properties 파일은 가장 낮은 우선순위를 가지기 때문에 환경 변수, CLI 인자 등으로 override 될 수 있다.

### Bootstrap properties

Spring Cloud에서 Bootstrap properties 파일은 Bootstrap context를 설정하기 위한 설정 파일이다.

그럼 boot strapcontext는 무엇인가? 이는 외부 source에서 설정값들을 가져오는 역할을 한다. 또한 `{cipher}*` 형태로 암호화되어 있는 property들을 복호화하는 역할도 한다.

Spring Cloud Commons - 1.1. The Bootstrap Application Context
: <https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#the-bootstrap-application-context>

Spring Cloud Commons - 1.10. Encryption and Decryption
: <https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#encryption-and-decryption>

설정 파일들을 가져오는 외부 source는 Spring Cloud Config Server를 의미한다.

### Bootstrap, the legacy way

<https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2020.0-Release-Notes#breaking-changes-1>

> Bootstrap, provided by spring-cloud-commons, is no longer enabled by default. If your project requires it, it can be re-enabled by properties or by a new starter.

2020.0.0 release train, Spring Cloud Config 3.0부터 Bootstrap은 legacy 방법이라고 설명하고 있다. 더이상 bootstrap이 기본적으로 활성화되지 않는다.

- Spring Cloud Config - 2.2.8.RELEASE 문서: <https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#config-first-bootstrap>
- Spring Cloud Config - Latest 문서: <https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#config-first-bootstrap>

## Conclusion

Bootstrap properties -> Bootstrap context -> Application properties -> Application context

위와 같은 계층 구조를 가지고 있다고 할 수 있다.

Spring cloud 3.0부터는 bootstrap을 별도로 enable 하지 않으면 bootstrap context는 설정되지 않을 것이다.
