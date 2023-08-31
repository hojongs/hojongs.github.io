---
layout: post
title: "Spring Boot: application properties 파일 override, 적용 우선순위 (+ Cloud Config)"
categories: [Spring Boot]
tags: [Spring Boot, Spring Cloud, Spring Cloud Config]
---

회사에서 팀원이 "Cloud config에서 정의된 property를 application.properties 파일로 override 할 수 있나요?"라고 질문했다. 가능하다는 것은 해봐서 알고 있었지만, Spring Cloud config 공식 문서에서 관련 레퍼런스를 본 적은 없었다. 왜 가능한걸까?

Spring Boot는 application properties file (profile이 있거나 혹은 없거나), 환경 변수, Java system properties, Cloud Config 등 다양한 방법으로 설정을 지원한다. Spring Boot에서 이것을 Externalized Configuration라고 말한다.

이 때, 하나의 property를(e.g. server.port property) 여러 개의 source에서 정의한다면, 어디서 정의된 property가 최종적으로 적용될까? 여기에는 우선순위가 필요하다. 특히, Cloud config를 통해 적용된 properties들은 서버 자체의 (jar 파일 내의) application.properties 파일을 통해 override 될 수 있을까?

> 문서를 잘못 읽어서 2시간 정도 헤맸는데, 그 내용도 추가했다 ㅠㅠ

## Cloud Config = Externalized Configuration

property override 관련 내용이 Spring Cloud Config 공식 문서에 있지는 않았다. 그 대신 Spring Cloud Config와 Spring Boot가 어떤 관계로 연결되는지 알아야했다.

### Externalized Configuration

구글링을 하다가 StackOverflow에서 Spring Boot의 Externalized Configuration 키워드에 대해 알게 되었다. 해당 페이지에서는 spring.profiles.active에 여러 개의 profile이 주어졌을 떄, **어떤 profile이 최종적으로 적용되는지**에 대한 Q&A가 있었다.

<https://stackoverflow.com/a/62222879/12956829>

> The last definition wins.

Spring Boot: Externalized Configuration: <https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config>

Spring Cloud Config 문서 첫 줄을 보면, `externalized configuration`가 언급된 것을 볼 수 있다. Spring Cloud Config와 Spring Boot를 연결해주는 관계인 것이다.

> Spring Cloud Config provides server-side and client-side support for externalized configuration ...

<https://docs.spring.io/spring-cloud-config/docs/3.1.5/reference/html/>

Spring Boot 문서를 참고했을 때, Config data files(properties 혹은 yaml 파일들)에 대하여 아래와 같이 언급되어 있다:

> Config data files are considered in the following order:
> 1. Application properties packaged inside your jar (application.properties and YAML variants).
> 2. Profile-specific application properties packaged inside your jar (application-{profile}.properties and YAML variants).
> 3. Application properties outside of your packaged jar (application.properties and YAML variants).
> 4. Profile-specific application properties outside of your packaged jar (application-{profile}.properties and YAML variants).

> resources 디렉토리에 있는 `application-dev.properties`와 같은 파일들은 Profile-specific application properties들이다. resources 디렉토리 안에 있는 properties 파일들은 jar 파일에 함께 package되지만 externalized configuration이라고 부른다. (애초에 모든 configuration들이 externalized 되어있다.)

사례 별로 살펴보면 아래와 같다:

- application.properties vs application-{profile}.properties: `application-{profile}.properties` 파일이 override한다.
- application.properties (서버 자체 정의, 즉 jar 파일 내부에 존재) vs application.properties (Cloud config에서 import, 즉 jar 파일 외부에 존재): **jar 외부인 Cloud config에서 import된 application.properties 파일이 override된다.**
  > 여기서 삽질을 한참 했는데, 기본적으로 Cloud Config Server에 정의된 application.properties 파일이 client-side local properties보다 더 높은 우선순위를 갖는다. 뒤에서 override-none property와 함께 더 설명한다.

아래는 혹시 몰라서 적어둔다.

> It is recommended to stick with one format for your entire application. If you have configuration files with both .properties and .yml format in the same location, .properties takes precedence.
> 
> 번역: `.properties` 혹은 `.yml` 파일  포맷 하나만 사용할 것을 권장하지만, 혹시 둘다 존재하면 `.properties` 포맷의 우선순위가 더 높다.

### Cloud config property가 override 하지 않게 (Local property가 override되게)

앞에서 언급한 "삽질"이 여기이다. 본인의 예상과 반대로, cloud config에서 정의된 property들이 기본적으로 더 높은 우선순위를 가지고 있었다. 그럼 이 우선순위가 어떻게 바뀐걸까? 별도의 설정이 있었던 것이다.

> You can change the priority of all overrides in the client to be more like default values, letting applications supply their own values in environment variables or System properties, by setting the `spring.cloud.config.overrideNone=true` flag (the default is false) in the **remote repository**.

<https://docs.spring.io/spring-cloud-config/docs/3.1.5/reference/html/#property-overrides>

`spring.cloud.config.override-none=true`으로 사용해도 된다.

본인만 헷갈린 것일 수도 있겠지만, 위 property는 **remote repository**에 설정해야 한다. 즉, cloud config server의 backend에 있는 git 저장소의 properties 파일들에 설정해야 한다. (cloud config client도 아닌, cloud config server도 아닌)

> spring.cloud.config.overrideNone=true: Override from any local property source.

<https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#overriding-bootstrap-properties>

아래 페이지에서는 `spring.cloud.config.override-none` 외에 `spring.cloud.config.*`를 포함한 모든 Spring Cloud의 properties들의 목록을 확인할 수 있다.

> Flag to indicate that when {@link #setAllowOverride(boolean) allowOverride} is true, external properties should take lowest priority and should not override any existing property sources (including local config files). Default false.

<https://docs.spring.io/spring-cloud/docs/2021.0.5/reference/html/configprops.html>

아래 페이지에서는 상세한 예제와 함께 작성되어 있다. `baeldung.properties` 파일에 해당 `spring.cloud.config.overrideNone=true`를 추가하는 것을 알 수 있다.

> 6.2. Enabling the Overriding Capability
> 
> On the server side, we need to indicate that property overloading is possible. Let's modify our baeldung.properties file in this way:
> 
> `spring.cloud.config.overrideNone=true`

<https://www.baeldung.com/spring-cloud-config-remote-properties-override>

> 문서에 override-none, overrideNone 두 개가 혼재되어 있으나, 어느 쪽이든 적용 가능할 것이다.

## Conclusion

properties 파일들과 같은 여러 개의 configuration이 주어졌을 때 어떤 것이 더 높은 우선순위를 가지는가? 아래 페이지를 참고하자.

> Spring Boot: Externalized Configuration: <https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config>

Cloud config에서 import된 properties는 기본적으로 jar 파일 내부에 존재하는 properties 파일보다 더 높은 우선순위를 가진다. 그러나, remote repository에 `override-none` property를 `true`로 설정함으로써, 최하위의 우선순위를 가지도록 변경할 수 있다.

## References

- Spring Cloud properties: <https://docs.spring.io/spring-cloud/docs/2021.0.5/reference/html/configprops.html>
- 그 외 본문 내 링크들
