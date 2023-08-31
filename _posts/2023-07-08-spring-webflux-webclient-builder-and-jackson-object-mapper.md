---
layout: post
title: "Spring WebFlux WebClient Builder bean 사용 이유 (ObjectMapper bean)"
categories: [Spring WebFlux]
tags: [Spring WebFlux, WebClient, Jackson]
---

## 이전 상황

WebClient를 build할 때 `WebClient.builder()`를 직접 호출해서 WebClient 인스턴르를 생성하고 있었다.

## 문제 상황

이 경우 Spring bean으로 등록된 `ObjectMapper`가 WebClient에 사용되지 않는다. 이 자체로는 문제를 유발하지 않지만, ObjectMapper bean의 설정을 변경해도 반영되지 않기 때문에 혼란을 유발할 수 있다. 이러한 문제는 디버깅을 어렵게 만든다.
- Spring boot에서 제공하는 `spring.codec.max-in-memory-size` property 설정이 적용되지 않는 문제도 있다. 별도 설정으로 사용하고 있었는데, 중복 설정이기 때문에 혼란이 있었다.
- `MetricsWebClientCustomizer`가 적용되지 않아서 `http.client.requests` 등 metric이 보이지 않는 문제. (수동 적용해줘도 됨)

## 변경

`WebClient.Builder` bean을 사용하여 WebClient를 build하도록 변경했다.
- `WebClient.Builder` bean?: Spring Boot는 `WebClient.Builder` bean을 제공해준다. 이 builder bean을 통해 WebClient
    - <https://docs.spring.io/spring-boot/docs/2.5.14/reference/htmlsingle/#features.webclient>

## 추가 문제 상황

별도로 설정한 ObjectMapper bean에 JavaTimeModule이 등록되지 않았다. DTO의 LocalDate 필드를 deserialize하지 못하는 문제가 발생했다.
- Why?: 우선 JavaTimeModule 모듈을 등록해서 해결하긴 했는데, 기존에는 왜 문제가 발생하지 않은걸까?
- WebClient.builder()를 직접 호출하면 당연히 Spring bean을 사용할 수 없다. 하지만 WebClient는 ObjectMapper가 필요하다. Jackson2JsonDecoder에서 요구하기 때문이다.
- 따라서 Jackson2JsonDecoder는 내부적으로 default ObjectMapper 인스턴스를 만들어서 사용한다: `Jackson2ObjectMapperBuilder.json().build()`
- 위 코드 내부적으로 `registerWellKnownModulesIfAvailable()`라는 메소드를 호출하는데, 여기서 JavaTimeModule 등의 모듈들이 등록된다. 이것이 기존에 에러가 발생하지 않았던 이유다.
- Spring boot는 `JacksonCodecConfiguration` 클래스에서 ObjectMapper bean을 사용하여 Jackson2JsonDecoder/Jackson2JsonEncoder 인스턴스를 생성하여 WebClient의 codec를 customize한다.

## Learning

- WebClient.Builder bean을 사용해야 ObjectMapper bean을 사용하여 JSON을 en/decode한다. (otherwise, 내부적으로 ObjectMapper를 생성해서 사용)
- `ObjectMapper()`와 같이 생성자를 통해  ObjectMapper 인스턴스를 생성할 수도 있지만, `Jackson2ObjectMapperBuilder.json().build()` 메소드를 통해 생성할 수도 있다. (일부 기본 설정들이 제공된다.)
- WebClient의 codec config 계층 구조:
    - WebClient -> ClientCodecConfig -> (Jackson2Json)En/Decoder -> ObjectMapper
