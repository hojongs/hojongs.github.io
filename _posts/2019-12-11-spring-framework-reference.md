---
layout: post
title: Spring Framework Reference Reading
tags: [Spring Framework, Java, Kotlin, IoC, 
  Dependency Injection,Bean]
comments: true
---

# Spring Framework Documentation
    
## Overview

## Core
### 1. The IoC Container
#### 1.1. Introduction to the Spring IoC Container and Beans
#### 1.2. Container Overview
##### 1.2.1. Configuration Metadata

* 앞 다이어그램이 보이듯, Spring IoC 컨테이너는 설정 메타데이터의 형태를 소비한다.
* 이 설정 메타데이터는 스프링 컨테이너에게 오브젝트들을 인스턴스화, 설정, 조립하는 방법을 표현한다.

* 설정 메타데이터는 전통적으로 간단하고 직관적인 XML 포맷으로 제공된다
* 이것은 이 챕터의 대부분이 핵심 개념과 IoC 컨테이너의 특징들을 전달하기 위해 사용한다

> XML 기반 메타데이터가 설정 메타데이터의 유일한 형태는 아니다
> IoC 컨테이너 자체는 설정 메타데이터가 실제로 작성되는 포맷과 완전히 분리되어 있다
> 오늘날, 많은 개발자들은 Java 기반 설정을 사용한다

...

== Java 기반 컨테이너 설정

* 이 섹션은 어노테이션을 컨테이너 설정을 위해 사용하는 방법을 다룬다

=== 기본 개념 : @Bean @Configuration

* 새로운 스프링 자바 설정 지원의 중심 아티팩트는, @Configuration-annotation된 클래스들과 @Bean-annotation된 메소드들이다

* @Bean 어노테이션은 메소드가 새로운 오브젝트를 인스턴스화, 설정, 초기화함을 나타낸다
* 이 오브젝트는 IoC 컨테이너에 의해 관리될 것이다
* `<beans/>` XML 설정과 친숙한 자들을 위해, @Bean 어노테이션은 `<bean/>` 요소와 같은 역할을 한다
* @Bean 어노테이션된 메소드는 @Component에 사용할 수 있지만, 대부분 @Configuration beans에 사용된다

* @Configuration으로 어노테이션된 클래스는 이것의 주요 목적이 bean 정의의 소스임을 나타낸다
* 또한, @Configuration 클래스들은 bean 사이 의존성이 정의될 수 있게 한다
  * 이는 클래스 내의 다른 @Bean 메소드를 호출함으로써 정의된다
* 가장 간단한 @Configuration 클래스는 다음과 같다

```
@Configuration
class AppConfig {

    @Bean
    fun myService(): MyService {
        return MyServiceImpl()
    }
}
```

앞선 AppConfig 클래스는 아래 `<beans/>` XML과 대응된다

```
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

Full @Configuration vs "lite" @Bean mode?
`TODO`

=== Instantiating the Spring Container by Using AnnotationConfigApplicationContext

`TODO`

==== Simple Construction

`TODO`

==== Building the Container Programmatically by Using register(Class<?>…​)

`TODO`

==== Enabling Component Scanning with scan(String…​)

`TODO`

==== Support for Web Applications with AnnotationConfigWebApplicationContext

`TODO`




## Testing

## Data Access

## Web Servlet

## Web Reactive

## Integration

## Languages