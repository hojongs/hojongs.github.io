---
layout: post
title: "[TIL] Istio distributed tracing"
categories: [Istio/Envoy]
tags: [Istio, TIL, distributed tracing]
---

https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/

## Istio Distributed Tracing Overview

Distributed tracing: service mesh에서 여러 서비스를 경유하는 요청을 트래킹하기 위해서 필요함
이것은 시각화를 통해 request latency, 순차적/병렬적 요청 여부를 더 깊이 이해할 수 있게 도와줌

Istio는 Envoy의 distributed tracing 기능을 활용함

https://www.envoyproxy.io/docs/envoy/v1.12.0/intro/arch_overview/observability/tracing

## Trace context propagation

Istio proxy가 자동으로 span을 보내지만 이 span들을 단일 trace로 join하려면 추가 정보가 필요함
애플리케이션들은 이 정보들을 HTTP header에 전파해야함

요청으로 들어온 header들을 수집하고 이 요청 중에 외부 요청이 발생하면 그 헤더들을 전달해야 함

어떤 헤더를 사용할 것인지는 사용할 trace backend에 따라 다름 (e.g. Zipki, Jaeger, ...)

## 헤더

x-request-id: envoy 전용 헤더
Zipkin, Jaeger 등은 B3 header를 사용함

https://github.com/openzipkin/b3-propagation

- x-b3-traceid
- x-b3-spanid
- x-b3-parentspanid
- x-b3-sampled
- x-b3-flags

`productpage` 샘플 서비스에서는 OpenTracing 라이브러리를 사용하여 헤더들을 수집하고 있음

https://opentracing.io/

> 참고로 OpenTracing 프로젝트는 아카이빙되었고, OpenTelemetry 프로젝트로 이어짐
> OpenTracing, OpenCensus 프로젝트가 OpenTelemetry로 통합되었음
> https://medium.com/opentracing/opentracing-has-been-archived-fb2848cfef67

### 우리의 헤더

x-company-event-id: c605808e52b066de
x-company-trace-id: c605808e52b066de

## 우리 애플리케이션에서의 헤더 전파는?

Feign client 사용 중
spring-cloud-starter-sleuth 의존성 추가 시 SleuthFeignBuilder(BeanFactory, Client)를 사용하여 feign client를 build 할 수 있음

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html

## Spring Cloud Sleuth: Terminology

Span, Trace라는 용어는 Dapper paper에서 가져온 것임

https://research.google/pubs/pub36356/

## 4.1 Context Propagation

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html#features-context-propagation

default header foramt은 B3

trace id 외에 다른 속성들도 전파될 수 있음 (Baggage)

`spring.sleuth.propagation.type` property를 설정하여 다른 propagation type을 사용할 수 있음

Brave의 경우 AWS, B3, W3C 타입을 지원함

https://github.com/openzipkin/brave

> Spring Cloud Sleuth는 Brave로 구현되었음

## Baggage (Brave and MDC)

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html#features-baggage
https://github.com/openzipkin/brave/tree/master/brave#baggage

MDCScopeDecorator
- context로 MDCContext를 사용함
- MDCContext: CorrelationContext interface의 구현체. update 시 MDC.put() & MDC.remove()를 호출함

BaggageFields
BaggageFields.TRACE_ID
BaggageFields.PARENT_ID
BaggageFields.SPAN_ID
BaggageFields.SAMPLED

spring-cloud-sleuth-autoconfigure의 BraveBaggageConfiguration.correlationScopeDecorator()
-> MDCScopeDecorator.newBuilder()를 통해 bean 생성
  -> CorrelationScopeDecorator.Builder() 호출
    -> traceId, spanId를 추가함 (parentSpanId는 추가하지 않음)
-> BraveBaggageConfiguration에서 `spring.sleuth.baggage.correlation-fields` property의 field들을 추가
  -> property에 parentId라는 이름을 추가하는 것과, SingleCorrelationField.create(BaggageFields.PARENT_ID)는 다름

## Log integration

https://docs.spring.io/spring-cloud-sleuth/docs/current-SNAPSHOT/reference/html/project-features.html#features-log-integration

## Spring MVC, WebFlux support

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html#sleuth-http-server-integration

MVC: `org.springframework.cloud.sleuth.instrument.web.servlet.TracingFilter`
WebFlux: `org.springframework.cloud.sleuth.instrument.web.TraceWebFilter`

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html#sleuth-http-server-webflux-integration

## Istio ingress gateway

Istio ingrses gateway: https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/
K8s ingress: https://istio.io/latest/docs/tasks/traffic-management/ingress/kubernetes-ingress/
Istio Gateway: https://istio.io/latest/docs/concepts/traffic-management/#gateways
K8s Gateway: https://gateway-api.sigs.k8s.io/api-types/gateway/

## Etc

> ingress-gateway의 log는 어디서 확인할 수 있지?
> envoy-access-log-* - service:istio-ingressgateway

https://github.com/spring-cloud/spring-cloud-sleuth/wiki/Spring-Cloud-Sleuth-3.0-Migration-Guide#parentid-and-sampled-spanexportable-mdc-fields-names-are-no-longer-set

> parentId and sampled (spanExportable) MDC Fields names are no longer set
For performance reasons, we no longer set the following fields by default:

## Sleuth WebClient

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html#sleuth-http-client-webclient-integration

TraceExchangeFilterFunction

## Client config props

spring.sleuth.feign.enabled -> OpenFeign
기존 코드: SleuthFeignBuilder를 사용해 Feign.Builder prototype bean

## Sleuth features

https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/htmlsingle/spring-cloud-sleuth.html#project-features

- Context propagation


