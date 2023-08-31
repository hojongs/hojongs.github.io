---
layout: post
title: "Spring Cloud Config: 사용법 소개"
categories: [Server development]
tags: [Spring Cloud, Spring Cloud Config, Microservices]
---

Spring Cloud Config는 분산 시스템에서 통합 설정을 중앙화하여 관리할 수 있게 해주는 시스템이다. 보통 여러 개의 Microservice들이 모여 하나의 Application을 구성하는 MSA에 적합한 도구다. 간단한 사용법을 살펴보자.

아래 공식 문서의 Quick start 섹션을 참고헀다.  
<https://docs.spring.io/spring-cloud-config/docs/4.0.0/reference/html/>

> 이 문서는 client-side 사용법을 중점적으로 다룬다.

## What is Spring Cloud Config? How to use it?

Spring Cloud Config의 핵심 포인트들에 대해 먼저 이해하고 가자. 어느 use case들에 사용해야 하는지도 생각해보자.

> Spring Cloud Config provides server-side and client-side support for externalized configuration in a distributed system. With the Config Server, you have a central place to manage external properties for applications across all environments.

위에서 언급한대로, Cloud Config는 분산환경에서 사용하기 위한 독립된 설정 서버다.

> so they fit very well with Spring applications but can be used with any application running in any language.

Spring 계열이다보니 Spring 서버에서만 사용할 수 있다고 오해하기 쉬운데, **Non-Spring 서버들도 Spring Cloud Config를 사용할 수 있다.**

> As an application moves through the deployment pipeline from dev to test and into production, you can manage the configuration between those environments and be certain that applications have everything they need to run when they migrate

Enterprise 서버들은 개발 환경부터 production 환경까지 여러 환경들이 있다. 보통 이런 환경들은 공유하는 공통 설정들이 있다. "여러 분산 서비스들이" "여러 환경에 걸쳐" 설정을 공유하는 케이스에도 Cloud config가 매우 유용하다.

## Cloud Config server API 호출 방법

Spring Cloud Config는 Server와 Client로 나뉜다. 예제 서버는 아래 주소에 있다.

Sping Cloud Config example source code: <https://github.com/spring-cloud/spring-cloud-config/tree/v4.0.0/spring-cloud-config-server>

example의 config 저장소: <https://github.com/spring-cloud-samples/config-repo>

Cloud config server를 배포한 후, 아래와 같이 요청을 보내면 응답을 받을 수 있다.

```sh
curl http://localhost:8888/foo/development
```

Cloud config server는 응답에 "propertySources"를 포함하는데, 이 source들은 기본적으로 git을 backend 저장소로 사용한다. git 저장소의 위치는 cloud config server의 아래 property를 통해 정의할 수 있다.  
`spring.cloud.config.server.git.uri`

Cloud config server는 아래와 경로 API endpoint들을 제공한다.

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

공식 문서에 나와있듯이, 호출 예시는 다음과 같다.

```shell
curl localhost:8888/foo/development
curl localhost:8888/foo/development/master
curl localhost:8888/foo/development,db/master
curl localhost:8888/foo-development.yml
curl localhost:8888/foo-db.properties
curl localhost:8888/master/foo-db.properties
```

편의에 따라, properties format 또는 yml format으로 응답값을 받을 수 있다.

## Cloud config client 설정 방법

보통은 Spring boot 서버가 cloud config client가 될 것이다. Spring 서버에 아래 의존성을 추가하자.

```kotlin
// Gradle Kotlin DSL
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-config")
}
```

별도 설정을 추가하지 않으면 `localhost:8888`을 cloud config server 주소 기본값으로 사용할 것이다. 아래 설정을 추가하자. cloud config server의 주소가 `myconfigserver.com`일 경우이다. (아래 설정은 필수다)

```properties
## application.properties
spring.config.import=optional:configserver:http://myconfigserver.com
```

> `optional:` prefix를 지우면 cloud config 요청 실패 시 서버 실행이 실패한다.  
> <https://docs.spring.io/spring-cloud-config/docs/4.0.0/reference/html/#_spring_cloud_config_client>
>> Removing the optional: prefix will cause the Config Client to fail if it is unable to connect to Config Server

application name의 기본값은 `application`이다. 바꾸려면 아래와 같이 설정하자.
하나의 서버에서만 사용하는 설정은 Cloud config에 올릴 필요가 없다. 서버 자체 application.properties에 정의하면 되기 때문.

아래 설정은 전체 서비스 중 일부 애플리케이션들만 공유하는 설정이 있을 때 유용하다. 예를 들어 `myapp`이라는 application name을 가진 서비스들만 공유하는 설정이 있을 경우이다. 의도치 않게 다른 서비스에서 해당 설정을 사용하는 경우를 방지할 수 있다.

그런 케이스가 아니라면 따로 설정하지 않아도 된다.

```properties
spring.application.name=myapp
```

### Legacy bootstrap way

Spring cloud config 옛날 버전 시절에 설정된 서버의 경우, cloud config client가 bootstrap 방식으로 설정되어 있을 수 있다. 자세한 내용은 아래 문서를 참고하면 된다.

이 방식은 bootstrap.properties 파일을 통해 cloud config client를 설정한다.

<https://docs.spring.io/spring-cloud-config/docs/4.0.0/reference/html/#config-first-bootstrap>

Spring Cloud Bootstrap에 대한 자세한 내용은 [여기로](/posts/spring-cloud-what-is-bootstrap/)

### Non-Spring cloud config client

이건 특별할 것이 없다. 앞서 언급한대로, Cloud config server API path에 맞춰 요청 시 cloud config를 조회할 수 있다.
조회한 값을 해당 Non-Spring application에서 사용하면 된다.

이는 다음과 같은 경우에 유용하다. Spring application에 의존하지 않는 테스트 코드를 작성하거나, 스크립트 파일을 작성 시 (어떤 언어로 작성된 스크립트든) cloud-config의 값을 조회해서 활용할 수 있다.

## Conclusion

틀린 내용이나 궁금한 내용이 있으면, 댓글로 남겨주시길.
