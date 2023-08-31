---
layout: post
title: "[TIL] Kotlin Default Value와 Jackson ObjectMapper"
categories: [Kotlin]
tags: [TIL, Kotlin, Jackson]
---

```
JSON parse error: Cannot map `null` into type `boolean` (set DeserializationConfig.DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES to 'false' to allow); nested exception is com.fasterxml.jackson.databind.exc.MismatchedInputException: Cannot map `null` into type `boolean` (set DeserializationConfig.DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES to 'false' to allow)
```

```
[Source: (org.springframework.util.StreamUtils$NonClosingInputStream); line: 1, column: 288] (through reference chain: package.dto.MyDto["myField"]->java.util.ArrayList[1]->package.vo.MyVO["value"]
```

myField[1].value 필드의 값이 null로 들어왔음

FAIL_ON_NULL_FOR_PRIMITIVES 설정을 disable 하는 방법으로도 해결할 수 있지만, 더 위험함

Kotlin에서는 필드에 default value를 설정할 수 있음

하지만 null 값이 전달됐을 때, default value가 아니라 null 값이 들어감 -> 에러 발생

Kotlin DTO에 default value를 사용하지 말자
