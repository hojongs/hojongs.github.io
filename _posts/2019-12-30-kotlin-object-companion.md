---
layout: post
title: Kotlin Object, Companion Object, Anonymous Object
redirect_from:
  - /kotlin-object-companion/
categories: [Server development]
tags: [Kotlin]
---

- 아래 Kotlin 공식 문서에서 관련 내용을 찾을 수 있었다
    - [Objects and companion objects](https://kotlinlang.org/docs/tutorials/kotlin-for-py/objects-and-companion-objects.html)

# Object

- Kotlin에서는 language-level에서 singleton pattern이 지원된다
- class 대신, `object` 키워드를 사용하여 선언하면 그 object는 singleton이 된다

```kotlin
    object CarFactory {
        fun makeCar(...) { ... }
    }
```

# Companion Object

- singleton instance보다는 class의 static method를 선언하고 싶을 때는, `companion object`를 활용할 수 있다

```kotlin
    class Car {
        // 이름은 optional
        // companion object {
        companion object Factory { 
            fun makeCar(...) { ... }
        }
    
        companion object {
            fun makeCar(...) { ... }
        }
    }
```

- companion object는 singleton이지만, 멤버들은 class name을 통해 접근 될 수 있다
- companion object name을 통해서도 접근할 수 있다

```kotlin
    Car.makeCar(...)
    Car.Factory.makeCar(...)
```

- companion object는 상속도 가능하다

```kotlin
    companion object Factroy : FactoryBase
```

## `@JvmStatic`

- Java static method로 만드려면, member에게 `@JvmStatic` annotation을 붙이면 된다

- A class는 오직 하나의 companion object를 가질 수 있고, 이들은 중첩될 수 없다

## overridable Class-level function in a subclass

- companion object의 멤버들은 class name을 통해서만 접근 가능하고, instance를 통해서는 접근 불가능하다
- Kotlin은 overridable class-level function in a subclass을 지원하지 않는다
    - subclass에서 companion object를 재선언하면, base class의 것은 가려진다

<br>

- overridable class-level function이 필요하다면 (instance를 통해 호출 시 override된 것이 호출되게 하고 싶다면)
- `open` function사용하라
- 이러한 구현이 가능하긴 하지만 inconvenient임을 인지할 것

> `open` 은 subclassing 또는 overriding을 가능하게 한다
- [Keywords and Operators](https://kotlinlang.org/docs/reference/keyword-reference.html)

# Object Expression

- java에서 interface를 parameter로 가지는 function을 호출할 때, object expression을 사용할 수 있다
- 즉, anonymous object를 다음과 같이 선언할 수 있다

```kotlin
    start(object : Vehicle { override ... }
```
