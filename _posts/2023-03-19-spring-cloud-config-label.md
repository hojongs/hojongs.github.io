---
layout: post
title: "Spring Cloud Config: Label 사용하여 특정 Git branch의 config 사용하기"
categories: [Spring Cloud]
tags: [Spring Cloud, Spring Boot]
---

일반적으로 엔터프라이즈 서버를 운영한다면, 애플리케이션 서버의 환경은 development(dev)와 production(prod) 환경이 분리되어 있을 것이다.

동일한 맥락에서 Cloud config server를 사용할 때, dev 환경과 prod 환경의 설정 파일들도 분리되어 있는 것이 안전하다.

이를 구현하기 위해 Spring Cloud Config의 label 기능이 필요하다.

## Git backend as config source

<https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_git_backend>

Git 저장소로 설정 파일들을 관리하는 환경을 전제로 서술하겠다.

> This repository implementation maps the {label} parameter of the HTTP resource to a git label (commit id, branch name, or tag).

아래는 label로 사용할 git branch의 이름에 `foo/bar` 처럼 `/`를 포함하고 있을 경우를 위한 설명이다. cloud config server endpoint 호출 시 url path와 섞여서 모호해지기 때문에 `/`를 `(_)`로 대체하여 `foo(_)bar`와 같이 요청해야 한다.

> If the git branch or tag name contains a slash (/), then the label in the HTTP URL should instead be specified with the special string (_) (to avoid ambiguity with other URL paths). For example, if the label is foo/bar, replacing the slash would result in the following label: foo(_)bar. The inclusion of the special string (_) can also be applied to the {application} parameter. If you use a command-line client such as curl, be careful with the brackets in the URL — you should escape them from the shell with single quotes ('').

label은 `spring.cloud.config.label` proproperty를 통해 설정할 수 있다

```properties
spring.cloud.config.label=foo(_)bar
```

<https://docs.spring.io/spring-cloud/docs/2021.0.5/reference/html/configprops.html>

## Separate dev, prod environments by label

Config 저장소에 develop, master 브랜치가 있다고 가정한다.

```properties
# application-dev.properties
spring.cloud.config.label=develop
```

```properties
# application-prod.properties
# master는 기본값이므로 생략 가능
spring.cloud.config.label=master
```

이제 변경사항을 develop 브랜치에 먼저 push하여 dev 환경의 서버들이 의도대로 동작하는지 먼저 확인할 수 있다. develop 브랜치에 push한 변경사항은 prod 환경에 영향을 미치지 않는다.

dev 환경에서 테스트가 완료된 후 develop 브랜치의 변경사항을 master 브랜치에 merge한다. 이후에 배포되는 prod 애플리케이션 서버들도 이제 설정값의 변경사항에 영향받게 될 것이다. 정상적으로 테스트하였다면 dev와 동일하게 의도대로 동작할 것이다.

## Get config URL path

아래와 같이 앞에 `/{label}` path prefix를 추가해주면 된다.

`{cloud-config-server-baseurl}/{label}/{application}-{profile}.properties`

이 외에도 여러 가지 방법이 있는데, 아래의 이전 포스트를 참고하면 된다.

[Spring Cloud Config: 사용법 소개](/posts/spring-cloud-config-usage/)
