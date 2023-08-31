---
layout: post
title: "[TIL] Envoy access log: start_time 정렬 문제"
categories: [Istio/Envoy]
tags: [Istio, Envoy, Logging, Elasticsearch, Kibana]
---

네트워크 로그 분석을 위해 envoy access log를 남기고 Elasticsearch(ES)에 적재하고 있다.

이번 포스트에서는 Envoy access log의 정렬 key 문제에 대해 설명한다.

ES의 로그는 Kibana에서 조회하고 있는데 Kibana에서 로그 조회 시 기본적으로 `@timestamp` 필드로 정렬된다.

이 정렬 방식은 일반적인 상황에서는 잘 동작하지만 Envoy access log의 경우에는 조금 문제가 있다.

`@timestamp` 필드는 ES에 로그가 적재된 시간을 의미한다. 일반적으로 이 순서대로 로그를 보면 되지만 Envoy access log의 경우에는 상황이 다르다. 아래 그림을 참고하자.

![envoy-access-log-inbound-outbound-order](/assets/img/posts/envoy-access-log-inbound-outbound-order.png)

로그 분석 시에는 outbound -> inbound 순서대로 읽는 것이 편하지만 로그가 적재되는 시점은 반대이다.

하나의 트랜잭션에서 여러 개의 outbound->inbound 로그가 섞이면 순서가 뒤섞여 분석하기 더더욱 어려워진다.

## Solution

Envoy access log에는 start_time이라는 값이 있다. HTTP 통신의 경우 `Request start time including milliseconds.`를 의미한다.

<https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators>

Kibana UI에서 이 값으로 정렬해서 보면 되지만, 로그 조회 시마다 정렬을 변경해주는 건 매우 번거롭다. 이 문제는 Kibana index pattern의 설정 변경으로 해결할 수 있다.

### Kibana index pattern and time field

Kibana index pattern 생성 시 time field를 지정할 수 있는데, `@timestamp` 대신 `start_time`으로 지정하면 된다.

![kibana-index-pattern](/assets/img/posts/kibana-index-pattern.png)

Tip: time field 변경을 위해 index pattern을 제거 후 재생성해야 하는데, index id를 기존과 동일한 값으로 사용하자. saved object들이 이 index id에 의존하기 때문에 index id가 변경되면 saved object들이 깨지는 문제가 발생한다.

time field 변경 후 이제 `Time` 필드(sort key)에 start_time 필드의 값이 보이는 것을 확인할 수 있다.
