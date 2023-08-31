---
layout: post
title: "[TIL] What is LDAP, Definition (Lightweight Directory Access Protocol)"
categories: [Network]
tags: [TIL, Network, LDAP, Protocol]
---

https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol
https://www.redhat.com/ko/topics/security/what-is-ldap-authentication

Filter는 Spring framework가 아닌 Java Servlet([Baeldung])에 존재하는 interface다. 

## Filters



OncePerRequestFilter: 가장 익숙한 이 구현체는, filter가 각 request를 한 번만 처리하도록 보장하는 구현체다. 로직을 요약하면 아래와 같다

- request에 `already filtered` attribute가 이미 있으면 request를 다음 filter에게 넘긴다.
- else, request에 `already filtered` attribute를 설정하고 filter 로직을 수행한다.

코드를 보면 request에 를 설정


[Baeldung]: https://www.baeldung.com/java-servlets-containers-intro
