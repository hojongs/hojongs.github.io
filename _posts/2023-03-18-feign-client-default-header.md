---
layout: post
title: "Feign: Feign client에 Default header 설정하기 (Request interceptor)"
categories: [Feign]
tags: [Feign]
---

## Feign: Feign client에 Default header 설정하기

기존의 코드에서 번거롭게 되어있던 부분을 리팩토링했다.

Feign client를 통해 요청을 보낼 때마다 동일한 Authorization 헤더의 토큰값을 파라미터로 전달해야 했다.

이 토큰 값은 DI를 받아야 했기 때문에, 더 번거로웠다. 이 Feign client를 DI받는 클래스들은 모두 token도 어쩔 수 없이 함께 DI 받고 있었다.

feign client 3개를 DI 받는 클래스는 총 6개를 bean들을 DI 받아야 했다.

이 feign client는 GitHub REST API client였다.

## Feign request interceptor

<https://www.baeldung.com/java-feign-request-headers>

feign client build 시 request interceptor를 설정하여 해결할 수 있었다.

```kotlin
feignBuilder
    .requestInterceptor { requestTemplate ->
        requestTemplate.header("Authorization", "token $githubToken")
    }
```

## Multiple request interceptor

위 참고문서에 언급되어 있는 것처럼 requet interceptor는 여러 개를 설정할 수 있다.

하지만 interceptor들 사이에 순서는 보장할 수 없다고 한다.

Feign은 써볼수록 사용성이 아쉬운 것 같다. WebClient로 빨리 바꿔야겠다.
