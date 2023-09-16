---
layout: post
title: "jitter(Retry backoff): jitter 사용 이유 및 레퍼런스들"
categories: [jitter]
tags: [Retry Backoff, Network, Concurrency, Distributed system]
---

MSA 또는 MSA 환경이 아니어도 외부 서비스 또는 시스템 장애로 인해 외부 호출이 실패할 수 있다. 일부 서비스와 실패가 시스템 전체 장애로 번지기도 한다. 실패를 완전히 방지할 수는 없다. 하지만 탄력적인 시스템을 구축하여 일부 서비스가 실패하더라도 전체 시스템은 문제없이 동작하도록 할 수는 있다.

탄력적인 시스템을 구축하기 위해 활용할 수 있는 도구들로 timeout, retry, backoff가 있다. jitter도 그 중 하나, 즉 탄력적인 시스템을 구축하기 위한 도구이다.

이 포스트에서는 jitter와 관련된 레퍼런스 문서들을 제공하고 이에 대한 요약을 정리하였다.

## AWS: Timeouts, retries, and backoff with jitter

Reference: <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>

- 다른 서비스나 시스템을 호출할 때 실패가 발생할 수 있고, 탄력적인 시스템 구축을 위한 도구 3가지를 소개함: timeout, retry, backoff
- client의 너무 긴 대기 시간으로 인한 리소스 부족 문제를 방지하기 위한 timeout, 일시적인 오류로 인한 실패를 극복하기 위한 retry, 호출받는 시스템의 과부하를 방지하기 위한 backoff.
- 추가적으로 여러 개의 retry 요청들이 동시에 도착하여 발생하는 문제를 방지하기 위한 jitter까지 소개함.
  - jitter는 오류의 원인이 과부하 또는 경합일 경우 retry로 인한 오류 재발생 확률을 줄여줌.
  - jitter는 재시도 뿐만 아니라 타이머나 정기 job, 지연된 작업에도 적용하여 traffic spike를 분산시킬 수 있음.

## AWS: Exponential backoff and Jitter

Reference: <https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/>

- 8년 전인 2015년에 작성된 글이지만 여전히 유용함
- OCC(Optimistic concurrency control, 낙관적 동시성 제어)는 여러 writer가 쓰기 손실 없이 하나의 entity를 안전하게 수정할 수 있는 테크닉임.
- 하지만 경쟁이 치열할 때 완료 시간 및 작엽량이 비례적으로 높아짐. backoff를 추가해도 문제가 해결되지 않음.
- 경합이 치열한 경우의 비교적 구체적인 사례로 OCC를 소개하고 jitter를 활용한 문제 해결 과정 및 결과를 그래프로 자세히 보여줌.

## Baeldung: Better Retries with Exponential Backoff and Jitter

Reference: <https://www.baeldung.com/resilience4j-backoff-jitter>

- resilience4j-retry 모듈에서 jitter 기능을 제공함.
- 이 모듈을 이용해 jitter를 구현하는 간단한 코드 예제를 보여줌.

## Additional References

https://stackoverflow.com/questions/46939285/why-is-random-jitter-applied-to-back-off-strategies
