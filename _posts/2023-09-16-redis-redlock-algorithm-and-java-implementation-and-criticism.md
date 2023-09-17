---
layout: post
title: "Redis: Redlock(Distributed lock) 알고리즘 및 Java 구현체, 그리고 비판"
categories: [Redis]
tags: [Redis, Java, Lettuce, Redisson, Spring Data Redis, Distributed lock]
---

Redis에서 제안한 Redlock(DLM implementation) 알고리즘을 이해하기 위해 Redis 공식 문서를 읽으면서 일부 내용을 정리했다.

## 용어집: Redlock

Reference: <https://redis.com/glossary/redlock/>

- Redlock에 대한 간단한 소개

## Redis를 사용한 분산 잠금

Reference: <https://redis.io/docs/manual/patterns/distributed-locks/>

- 기존 Redlock 구현체 목록을 소개하고, 더 안전하다고 생각되는 Redlock 알고리즘에 대해 설명한다.

### 내용 정리 (일부)

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
  1. 현재 시간을 가져옴 (millis unit)
  2. 각 Redis instance에서 lock 획득 시 자동 해제 시간보다 작은 timeout을 사용함 (e.g. 10s vs 5ms)
     이를 통해 너무 긴 blocking을 방지할 수 있음
  3. lock 획득에 소요된 시간을 계산하고, 이것이 lock 유효 시간보다 작으면 lock 획득으로 간주
  4. lock 획득 실패 시 모든 인스턴스 lock 해제 시도

Is the Algorithm Asynchronous?

- 모든 프로세스에 동기화된 clock은 없음
- 하지만 모든 프로세스의 로컬 clock은 auto-release 시간 대비 오차가 작음
- mutual exclusion rule을 정해야 함: step 3에서 획득한 lock validity time에서 일정 시간(n ms)을 뺀 시간동안 상호 배제 규칙이 보장됨 (clock drift를 보상하기 위해)

Retry on Failure

- lock 획득 실패 시 여러 클라이언트는 random delay 후 retry 해야 함
  -> 클라이언트들이 동시에 동일한 lock 획득 시도하지 않기 위해
  -> 이는 아무도 이기지 못하는 split brain 상태가 발생할 수 있음 (예시 필요)
  - 이 때 재시도는 각 instance가 아닌, instance set에 대한 retry를 의미하는 것으로 보인다
  - 예시: (master) instance가 5개 있고 각각 1~5라고 하자. 2개의 client A,B가 lock 획득을 시도한다.
    - 클라이언트 A는 instance 1,2, B는 instance 3,4의 lock을 획득했다.
    - 하지만 네트워크 파티션 등 오류로 인해 두 클라이언트 모두 instance 5로부터 lock을 획득하지 못하고 있다.
    - 두 클라이언트가 동일한 delay 후 재시도 시 위와 동일한 상황이 되어 또다시 둘다 lock을 획득하지 못할 가능성이 있다.
    - 하지만 random delay 후 시도하면, 여전히 instance 5가 응답하지 않을 때 하나의 클라이언트가 lock을 획득할 수 있다.
    - 예를 들면, 클라이언트 A가 instance 1,2,4로부터 lock을 획득하는 경우이다.
    - 참고로 [ref](https://s-core.co.kr/insight/view/redis-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EC%97%90%EC%84%9C-split-brain-%EC%83%81%ED%99%A9%EC%9D%98-%EB%8C%80%EB%B9%84%EC%B1%85/)에서 설명하는 split brain 경우는 random delay 후 retry로 해결되지 않는 경우로 보인다.

이후의 내용이 더 있으나 이 포스트에는 정리하지 않았다. 더 자세한 내용이 궁금하면 reference를 직접 참고하자.

Reference: <https://redis.io/docs/manual/patterns/distributed-locks/>

## Redlock 구현

- Redlock 알고리즘을 공부하는 목적으로 Redlock을 구현해보는 것은 좋은 공부가 될 수 있다
- 하지만 실제로 사용하기 위해 Redlock을 구현해야 한다면, 사용하려는 언어의 이미 구현된 라이브러리가 있는지 먼저 조사해보는 것이 좋다. 이미 존재한다면 [Reinventing the wheel](https://en.wikipedia.org/wiki/Reinventing_the_wheel) 할 필요 없다.
- 반대로 이미 옛날부터 Redlock을 구현하여 사용하고 있었고 잘 동작하고 있다면 그것을 라이브러리로 교체할 필요도 없다. (If your code works, don't touch it.)
- Java의 경우 [Lettuce](https://github.com/lettuce-io/lettuce-core)와 [Redisson](https://github.com/redisson/redisson)에 대해 살펴보자.
  - Lettuce는 low-level client로서 distributed lock과 같은 기능은 제공하고 있지 않다. ([lettuce-core#150](https://github.com/lettuce-io/lettuce-core/issues/150))
  - redisson은 RedLock 기능을 제공했었으나 deprecate 되었다. ([redisson#2669](https://github.com/redisson/redisson/issues/2669))
    - redisson의 RLock 구현체는 Redlock을 기반으로 하는 듯하지만 동일한 알고리즘은 아닌 것으로 보인다. ([redisson#956](https://github.com/redisson/redisson/issues/956))
  - Lettuce에서 언급한대로, Spring Data Redis도 한 번 살펴보았다. Spring Data Redis는 Lettuce보다 High-level client지만 (Redlock과 같은) Distributed lock을 구현하지 않기로 결정했다. 메커니즘이 매우 복잡하고 error-prone하기 때문이라고 한다. ([spring-data-redis#944](https://github.com/spring-projects/spring-data-redis/issues/944))
- 즉, Java의 경우 Lettuce를 사용한다면 Lettuce 기반으로 Redlock을 직접 구현해야 하는 것으로 보인다. Redisson의 RLock을 함께 사용하는 것도 대안이 될 수 있다.

## Redlock 알고리즘 비판

- Redisson issue에서 언급한 <https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html> 문서도 읽어보면 좋을 것 같다.
- 이 문서에서는 Redlock 알고리즘은 나쁜 선택(poor choice)라고 주장하고 있다.
- 이 알고리즘은 타이밍과 system clock에 대한 위험한 가정을 하고 있으며 이 가정이 위반되면 안전을 위반하고 있기 때문이다.
- (정확성보다) 효율성을 위해서라면 single-node locking 알고리즘을 사용하고
- 정확성을 위해서라면 Zookeeper와 같은 적절한 consensus system을 사용하라고 설명하고 있다.

- 그리고 바로 다음 날, Redis의 original 개발자 [antirez](https://github.com/antirez)는 반박 글을 작성하였다: [Is Redlock safe?](http://antirez.com/news/101)

## Conclusion

글을 작성하다보니 내용이 다양해졌다. 내용을 정리해보자.

- Redlock 알고리즘 동작방식에 대해 정리하였다.
- Java 구현체들에서 Redlock 관련 이슈들을 살펴보았다. (Lettuce, Redisson, Spring Data Redis)
- Lettuce를 사용한다면 Redlock은 직접 구현해야 할 것이다.
- Redlock 알고리즘에 대한 논란도 있다. Martin Kleppmann의 비판과 antirez의 반박 글이 있었다.
