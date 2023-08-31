---
layout: post
title: "[TIL] Kotlin: Data class copy, best practice"
categories: [Kotlin]
tags: [TIL, Kotlin, OOP]
---

copy 사용 시 장점과 단점?
- 장점: immutable object
- 단점: object 외부에서 필드값 직접 조작

best practice

> 다른 의견 환영!

- value object에 사용하기
- entity의 경우 copy 메소드는 object 내부에서만 사용하기.


