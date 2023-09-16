---
layout: post
title: "Redis Redlock (Distributed lock)"
categories: [Redis]
tags: [Redis]
---

<https://redis.io/docs/manual/patterns/distributed-locks/>

- 분산 락(Distributed lock)은 여러 프로세스가 상호배제(mutual exclusive) 방식으로 공유 리소스를 접근하는 경우 충돌을 방지할 수 있는 테크닉임
- Redis를 사용해 DLM(short for Distributed Lock Manager)을 구현하는 기존 라이브러리나 블로그들이 있지만 충분하지 않음
- 기존 방식들보다 안전한 DLM을 구현하는 Redlock 알고리즘을 제안함
- 장애 상황: master에서 lock을 획득하고, lock key가 복제되기 전에 master에 장애가 발생하면?

Correct Implementation with a Single Instance

- SET NX PX 명령어를 통해 lock을 획득함
- key가 존재하고 & value가 내가 기대한 값과 같을 때만 제거하도록 lua 스크립트로 구현함
  - 다른 클라이언트가 생성한 lock을 제거할 수도 있기 때문 (unsafe)
  - value에 저장할 random unique string은 /dev/urandom 또는 timestamp를 활용할 수 있음

The Redlock Algorithm

- 5개의 master가 있다고 가정함
- Step by step
  - 각 Redis instance에서 lock 획득 시 자동 해제 시간보다 작은 timeout을 사용함 (e.g. 10s vs 5ms)
  - 이를 통해 너무 긴 blocking을 방지할 수 있음
  - lock 획득에 소요된 시간을 계산하고, 이것이 lock 유효 시간보다 작으면 lock 획득으로 간주
  - lock 획득 실패 시 모든 인스턴스 lock 해제 시도

<https://redis.com/glossary/redlock/>
