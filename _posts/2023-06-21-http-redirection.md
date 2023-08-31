---
layout: post
title: "[TIL] HTTP Redirection with location header"
categories: [HTTP]
tags: [TIL]
---

bit.ly 서비스처럼 URL shorten 기능은 어떻게 구현할까?

HTTP Location header를 사용하면 된다.

Location header: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Location>

실제로 bit.ly에서 생성된 URL로 요청 시 HTTP response에 301 status code와 Location header를 사용하는 것을 볼 수 있다.

![bitly-redirection-1](/assets/img/posts/bitly-redirection-1.png)
![bitly-redirection-2](/assets/img/posts/bitly-redirection-2.png)

Location header으로 아예 다른 도메인으로 redirect될 수 있을까? 물론 위에서 본 바와 같이 가능하다.

HTTP Redirection: <https://developer.mozilla.org/ko/docs/Web/HTTP/Redirections>

## In-house URL shorten service

회사에서도 자체 URL 단축 서비스를 운영한다. 왜 bit.ly와 같은 public service를 사용하지 않을까?

이 서비스가 만들어질 당시에 있지 않아서 알 수 없지만, 아래와 같은 맥락들을 추측해볼 수 있다.

- 서비스 사용 비용이 발생한다. (In-house 서버 비용보다 클 경우에 유의미)
- 성능 요구사항을 보장받을 수 없거나 더 많은 비용을 지불해야 한다. (Latency, throughput, availability)
- 외부 서비스 의존성으로 인한 장애에 취약해진다.
- CTR 통계 등 다양한 요구사항이 필요하고, 이런 기능이 지원하지 않거나 비용을 지불해야 한다. (찾아보니 bit.ly URL은 expire되지 않고 데이터 분석 기능도 지원한다.)
- 필요한 요구사항이 해당 서비스에 존재하지 않는 기능일 수 있다.

> 결국 모두 돈으로 귀결되는 느낌...
>
> ref: <https://bitly.com/pages/pricing/v1>

서비스 규모가 작을 때는 SaaS를 사용하는 것이 합리적이지만 규모가 커질수록 지불해야 하는 비용이 더 커지고, in-house로 운영하는 것이 더 비용 합리적인 시점이 온다.

로그를 보니 우리 회사에서는 하루에도 수십 만 번씩 shorten API가 호출된다. (예상보다도 훨씬 많다)

bit.ly에서도 10,000/month 이상의 shroten 링크 생성은 enterprise plan에 들어간다.

이러한 이유로 우리는 자체 URL shorten 서비스를 운영하고 있다.

이 서비스 관련하여 작업할 일이 생겨서 이 서비스의 동작에 대해 알아보게 되었다.
