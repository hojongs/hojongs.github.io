---
layout: post
title: "Spring Cloud Config: logback.xml 파일 (plain text non-properties 파일) 제공하기"
categories: [Spring Cloud]
tags: [Spring Cloud, Spring Boot]
---

## Serving plain text files via Spring Cloud Config Servers

Spring Cloud Config 서버는 properties 파일 뿐만 아니라 그 외의 파일들도 제공할 수 있다.

예를 들면 logback.xml 파일들을 Spring Cloud Config Server에서 중앙화하여 관리할 수 있다.

<https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_serving_plain_text>

이러한 파일들을 조회할 때는 `/{application}/{profile}/{label}/{path}` url path를 통해 조화하면 된다.

e.g. `/application/dev/master/logback.xml`

이전 포스트와 마찬가지로 Git backend로 Spring Cloud Config Server를 구성했다면 `{path}`는 git 저장소 내에서 해당 파일의 path로 지정하면 된다.

config-repo의 root directory에 logback.xml 파일이 있다면 아래와 같이 요청하면 된다.

application=application, profile=dev, label=dev (branch), path=logback.xml

```sh
curl -v "http://localhost:8888/application/dev/main/logback.xml"
```
