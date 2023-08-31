---
layout: post
title: "JPA: optimisstic lock vs pessimistic lock"
categories: [JPA]
tags: [Concurrency, Database, JPA, Spring Data, Locking]
---

## What is optimistic locking?

JPA Spec 문서에서 낙관적 락에 대한 내용 일부를 인용하겠다.

> Optimistic locking is a technique that is used to insure that updates to the database data corresponding to the state of an entity are made only when no intervening transaction has updated that data since the entity state was read. This insures that updates or deletes to that data are consistent with the current state of the database and that intervening updates are not lost. Transactions that would cause this constraint to be violated result in an OptimisticLockException being thrown and the transaction marked for rollback.

**Optimistic lock, 낙관적 락은 엔티티를 읽은 후 다른 트랜잭션이 해당 엔티티를 수정하지 않은 경우에만 해당 엔티티 수정이 이루어지도록 하는 기술이다.**  

낙관적 락은 여러 트랜잭션이 동일한 엔티티에 접근하여 해당 엔티티를 동시에 수정할 때 유용할 수 있다.  
엔티티가 여러 트랜잭션에 의해 동시에 수정되어도 어떤 수정사항이 다른 수정사항에게 덮어씌워지는 등의 에러를 방지할 수 있다.
즉, 데이터의 무결성(integrity)을 보장할 수 있게 된다.

예를 들어, 2개의 트랜잭션 T1, T2가 있다고 하자.  
T1과 T2가 계좌(account) 엔티티에서 동시에 출금을 하려 한다.  
계좌에 처음에 10,000원이 저장되어 있다고 가정하자.  
이 때 T1, T2 각각 10,000원씩 출금한 후 잔액을 계좌 엔티티에 저장했다면, 둘 중 하나의 트랜잭션만 성공해야 할 것이다.

하지만 lock 없이 두 트랜잭션이 계좌 엔티티를 동시에 수정한다면 경우에 따라 두 트랜잭션이 모두 성공할 수도 있는 것이다.  

**이러한 문제를 해결하는 방법 중 하나가 낙관적 락이다.** 특히 낙관적 락은 write보다 read operation을 더 많은 사례(use case)에 적합하다.

낙관적 락은 version attribute를 사용하여 구현될 수 있는데, 좀더 자세히 살펴보자.

> NOTICE: This document is WIP!

## 낙관적 락 관련 문서들 찾아보기

낙관적 락(Optimistic Locking)에 관련된 문서를 찾아 보았다.

Baeldung: JPA Optimistic Locking

<https://www.baeldung.com/jpa-optimistic-locking>

위 문서 내에 비관적 락(Pessimistic Locking) 문서가 링크되어 있어서 함께 남겨놓는다.

Baeldung: JPA Pessimistic Locking

<https://www.baeldung.com/jpa-pessimistic-locking>

낙관적 락에 2가지 모드가 존재했다. OPTIMISTIC(READ) vs OPTIMISTIC_FORCE_INCREMENT(WRITE)  
위 문서만으로는 이해가 잘 되지 않아서 JPA 문서를 더 찾아 보았다.

## Spring Data JPA

<https://docs.spring.io/spring-data/jpa/docs/current/reference/html/>

Spring Data JPA 문서에서 locking 관련한 내용은 아래 뿐이었다.

<https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#locking>

> 관련 문서도 찾은김에 함께 남겨놓는다.
>
> https://www.baeldung.com/java-jpa-transaction-locks

JPA Repository에서 Lock 어노테이션에 대한 내용만 간단히 있었고, LockModeType의 타입별 스펙은 JPA 문서 자체를 찾아야 했다.

> 참고로 Spring Data JDBC 문서에는 optimistic locking 관련 내용이 있다.
>
> https://docs.spring.io/spring-data/jdbc/docs/current/reference/html/#jdbc.entity-persistence.optimistic-locking

Spring Data JPA 문서 첫 문단에 아래와 같이 언급되어 있었다.

> Spring Data JPA provides repository support for the **Jakarta Persistence API (JPA)**

일단 JPA 스펙 문서는 여기에.

Jakarta Persistence Specifications

<https://jakarta.ee/specifications/persistence/>

그런데 JPA는 Java Persistence API의 약자 아니었나? 무슨 차이지? 의문이 들어 잠시 찾아보았다.

### Java Persistence vs Jakarta Persistence?

JSR-338 Java Persistence 2.2 release specification document

<https://download.oracle.com/otndocs/jcp/persistence-2_2-mrel-spec/index.html>

<https://stackoverflow.com/a/60024749/12956829>

> The Java Persistence API (JPA), in 2019 renamed to Jakarta Persistence.

아하. 같은 것이고 이름이 바뀐 것이구나. 다시 낙관적 락 주제로 돌아가자.

## OPTIMISTIC vs OPTIMISTIC_FORCE_INCREMENT in JPA Specification

사실 우리네가 쓰고있는 건 JPA 2.2 spec이지만, 최신 문서인 3.1을 참고해보자.

### JPA Spec: 3.4.4.1. OPTIMISTIC, OPTIMISTIC_FORCE_INCREMENT

<https://jakarta.ee/specifications/persistence/3.1/jakarta-persistence-spec-3.1.html#a2100>



=== To be continued ===
