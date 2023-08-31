---
layout: post
title: "Spring WebFlux: Client disconnection 탐지 동작 방식"
categories: [Spring]
tags: [Spring, WebFlux, Spring MVC, Network, TCP/IP, HTTP, Netty, Servlet]
---

Keyword: Spring, WebFlux, Spring MVC, Network, TCP/IP, HTTP, Netty, Servlet

Spring WebFlux 서버에서 doOnCancel을 통해 로그를 남겼더니, client가 요청을 끊었을 때 로그를 남길 수 있는 것을 확인할 수 있었다. 이는 client에서 설정된 타임아웃 초과로 인한 연결 종료였다. Spring MVC에서는 이러한 로그를 남기는 것이 근본적으로 불가능하다. 그 이유가 무엇일까? 이와 관련해 Netty와 Servlet에 대해 알아보자.

## Spring WebFlux는 어떻게 Client disconnection을 탐지하나? Netty?

Spring WebFlux의 네트워크를 구현하는 코어 라이브러리 스택에 대해 알아보자.

- Spring WebFlux: Spring Framework과 integration된 최상위 레벨.
- Reactor: Spring WebFlux의 인터페이스이다. Mono, Flux와 같은 Reactor 타입을 사용한다.
- Reactor Netty: Reactor를 위한 Netty 어댑터이다. 하위 레벨에서 Netty를 사용하기 때문이다.
- Netty: Async Network communication library. Event loop 기반으로 동작한다. 하나의 스레드에서 여러 개의 커넥션을 처리할 수 있다.
- TCP/IP: Network level. Client disconnection 이벤트는 여기서부터 발생한다.

### Netty: channelInactive

> 아래부터는 아직 실제로 테스트해보지는 않았으므로 틀린 내용이 있을 수 있음.

Netty에 ChannelInboundHandler라는 인터페이스가 있다. 이는 channelInactive라는 메소드를 가지고 있고 연결이 끊어졌을 때 이 메소드가 호출되는 것으로 보인다.

ChannelInboundHandler: <https://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html>

channelInactive(): <https://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html#channelInactive-io.netty.channel.ChannelHandlerContext->

Netty는 event loop 기반이고 이와 같은 Network event들을 수신하여 콜백을 호출한다.

Reactor Netty에서는 이를 cancel signal로 해석하여 doOnCancel이 호출되도록 변환해줄 것이다.

이번에는 channelInactive() 메소드가 어떤 Network event로부터 호출되는지 알아봐야 한다. TCP/IP 레벨에서는 channelInactive라는 이벤트는 없다. Netty가 해석한 이벤트일 뿐이다.

### TCP/IP: Client disconnection

HTTP client에서 timeout이 초과했을 때 TCP/IP 수준에서는 FIN 패킷을 보낸다.

- TCP/IP: FIN 패킷 발생 이벤트
- Netty: channelInactive()호출
- Reactor Netty, Spring WebFlux: Mono/Flux의 상태에 따라 cancel 시그널을 보내 doOnCancel 호출됨
  > 이미 complete된 Mono/Flux는 doOnCancel이 호출되지 않을 것이다. 정상적인 HTTP 요청/응답 시나리오가 이에 해당함.  
  > HTTP/1.1 Keep-alive 초과로 인한 커넥션 종료도 정상적인 커넥션 종료이므로 doOnCancel이 호출되지 않는다.

FIN 패킷 외에 RST 패킷을 통해서 강제로 커넥션을 종료시키는 시나리오도 있을 수 있다. 이 케이스에도 Netty에서는 channelInactive 메소드가 호출될 것이다.

> RST 패킷은 보통 서버가 아니라 클라이언트 사이드에서 많이 볼 수 있을 것이다. 보안 정책에 의해 연결이 차단된 경우 클라이언트는 RST 패킷을 수신할 것이다. 이는 클라이언트에서 데이터를 보냈으나 이미 연결이 종료된 상황에 해당한다.

> 예를 들어 애플리케이션 서버에서 MySQL과 같은 Database에 데이터(쿼리 요청)를 보냈는데 이미 DB 서버에서 커넥션이 종료되었을 경우 RST 패킷을 수신할 수 있다. 애플리케이션 서버는 커넥션 오버헤드를 줄이기 위해 커넥션을 재사용하는데 (e.g. HikariCP) 커넥션을 재사용하려는 시점에 이미 DB 서버에서 커넥션이 종료되었을 수 있다.

## Spring MVC에서는 왜 안돼? Servlet?

Spring MVC도 TCP/IP 레벨에서는 Spring WebFlux와 동작이 거의 동일할 것이다. FIN 패킷을 수신할 것이다.

그런데 MVC에서는 이런 동작을 구현할 수 없다. 왜!일까?

> Spring MVC의 이러한 한계로 인해 처리가 오래 걸리는 Spring MVC 서버의 API의 경우, client에서 연결을 종료하더라도 서버에서는 여전히 API를 처리하고 있다.

> Istio/envoy와 같은 proxy 서버를 경유하여 통신하는 경우, 180초가 초과하는 경우 클라이언트 타임아웃이 180초 이상이어도 envoy proxy에 의해 연결이 종료될 수 있다. (response_flag=DC)

우리는 Spring MVC가 Servlet 기반으로 동작한다는 사실을 알아야 한다. Servlet의 역할에 대해서도 알아야 한다.

### Java Servlet?

Java Servlet는 웹 앱을 구현하기 위해 제공되는 Java의 API이다. Tomcat 등은 이를 실행하기 위해 제공되는 Servlet container이다.

웹 앱을 구현한다는 것이 의미하는 것은, Servlet이 HTTP 프로토콜 수준에서 API를 제공한다는 것이다.

즉 HTTP보다 더 low-level인 TCP 프로토콜의 이벤트에 대한 스펙(API)은 제공되지 않는다.

이를 통해 Servlet은 더 추상화되어 사용하기 쉽다는 편의성은 있지만 TCP 수준의 동작을 구현하기 어렵다는 제약이 생긴다.

따라서 Apache Tomcat과 같은 Servlet container를 기반으로 구현되는 Spring MVC는 근본적으로 TCP 수준에서 발생하는 Client disconnection 이벤트를 탐지하여 메소드를 호출하는 동작을 구현하기 어려운 것이다.

## Conclusion

Netty 기반인 Spring WebFlux에서는 client timeout 초과와 같이 HTTP(TCP) 연결의 비정상적인 종료 이벤트를 탐지하여 cancel signal을 수신할 수 있다. 

반면 Java HTTP Servlet 기반인 Spring MVC에서는 이러한 HTTP 연결 종료를 탐지하기 어렵다.

이러한 동작의 이유를 이해하기 위하여 내부 동작 방식을 이론적으로 분석해보았다.
