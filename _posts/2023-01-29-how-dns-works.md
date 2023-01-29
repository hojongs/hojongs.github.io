---
layout: post
title: "DNS 동작 방식 이해"
categories: [Server development]
tags: [DNS, Network]
---

웹브라우저에서 google.com 접속을 시도하면, DNS query를 통해 해당 도메인의 IP 정보를 획득하여 요청을 보낸다는 것은 대부분 알고 있을 것이다. 하지만, 그것은 DNS의 빙산의 일각일 뿐이다. DNS에 대해 조금더 깊이 알아보자.

# Motivation

DNS 장애는 꽤나 무서운 일이다. 장애가 한 번 발생되면 DNS 레코드가 수많은 네임 서버들에 전파되고 캐시되기 때문에 장애가 해결될 때까지 지켜보고만 있어야할 수도 있다. 어쨌든 장애 조치는 취해야 하는데, 조치를 취하려면 DNS에 대한 제대로된 이해가 필요하다. 예를 들면 이런 것들이다.

- DNS 쿼리는 어느 네임 서버로 요청을 보내는가?
- `Authoritative, Non-authoritative server`란 무엇인가?
- DNS 레코드의 TTL이란 무엇인가?
- DNS SOA 레코드의 역할은 무엇인가?
- DNS query의 negative cache란 무엇인가?
- nslookup, dig, whois command의 사용법 및 결과값 이해

위와 같은 의문들을 해결해보고자 한다.

# What is DNS?

AWS에 DNS 기초에 대한 좋은 문서가 있었다.

<https://aws.amazon.com/ko/route53/what-is-dns/>

## DNS service types & How DNS works

일반적으로 스마트폰, PC에서 DNS query를 요청하면 어디로 요청을 보내는걸까? 먼저 위 문서에 언급된 DNS 서비스 유형에 대해 알아보자

- Authoritative DNS(신뢰할 수 있는 DNS)
  - 도메인에 대해 최종 권한을 가짐
  - Recursive DNS 서버에 IP 정보를 제공함
  - e.g. Amazon Route53
- Recursive DNS(재귀적 DNS)
  - **대부분 클라이언트는 authoritative DNS 서비스에 직접 쿼리하지 않음**
  - 대신 Resolver 또는 Recursive DNS라고 알려진 DNS 서비스에 연결하는 것이 일반적
  - Recursive DNS는 DNS 참조를 캐시할 수 있음
  - 캐시되어 있지 않으면 DNS 쿼리를 Authoritative DNS로 전달함

![AWS DNS diagram1](https://d1.awsstatic.com/Route53/how-route-53-routes-traffic.8d313c7da075c3c7303aaef32e89b5d0b7885e7c.png)
> ref: <https://aws.amazon.com/ko/route53/what-is-dns/>

`www.example.com` 도메인을 쿼리할 떄, 도메인은 계층화되어 있기 때문에 DNS resolver는 여러 개의 authoritative DNS로 쿼리를 보낸다. root nama server, .com TLD name server, `example.com` 도메인의 authoritative name server 순서대로 쿼리를 보낸다. 마지막 name server는 www.example.com 도메인의 A 레코드를 가지고 있으므로, IP 정보를 응답한다. 만약 A 레코드가 존재하지 않으면 NXDOMAIN을 응답한다.

즉, .com TLD name server는 example.com 도메인의 authoritative server의 도메인 주소를 알고 있고 이를 응답한다는 것이다.

## Domain name registry?

상위 name server는 하위 name server의 도메인 주소를 어떻게 아는 것일까? 예를 들어 설명하면 다음과 같다.

- `hojong.site` 도메인의 authoritative name server 중 하나가 `NS03.DOMAINCONTROL.COM`라고 하자
- `.site` TLD name server 중 하나는 `A.NIC.SITE`라고 하자.
- 이 때, `A.NIC.SITE` name server는 `hojong.site` 도메인의 authoritative server가 `NS03.DOMAINCONTROL.COM`라는 것을 알고 있다는 것이다.

이것은 Domain name registry를 통해 이루어진다고 한다. 자세한 내용은 다른 사이트 레퍼런스들로 대체한다.

- <https://en.wikipedia.org/wiki/Domain_name_registry>
- [DNS란 뭐고, 네임서버란 뭔지 개념정리](https://gentlysallim.com/dns%EB%9E%80-%EB%AD%90%EA%B3%A0-%EB%84%A4%EC%9E%84%EC%84%9C%EB%B2%84%EB%9E%80-%EB%AD%94%EC%A7%80-%EA%B0%9C%EB%85%90%EC%A0%95%EB%A6%AC/)

도메인의 name server는 어떻게 알 수 있느냐? nslookup, dig, whois 중 하나를 사용하면 된다

```sh
nslookup -type=NS site
dig site NS
whois site
```

# Conclusion

글이 너무 길어질까봐 이번 글은 여기서 마치고, 나머지 의문들에 대한 답들은 다른 글들에서 작성할 예정이다.

몇몇은 DNS에 대한 내용을 꽤 지루하게 느낄지도 모른다. 이해라기보다는 암기에 가까운 느낌이 들어서 그럴지도 모른다. (아니면 내가 재미없게 설명하거니...) 하지만 DNS는 네트워크 어디서든 항상 의존하고 사용하는 시스템이기 때문에, 동작에 대한 호기심과 답답함이 있었다. 그래서 한 번 제대로 정리해두고자 했다.

## Additional references

<https://ns1.com/resources/dns-propagation>
