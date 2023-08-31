---
layout: post
title: "Spring Response Encoding UTF-8, EUC-KR, KSC5601"
categories: [Spring]
tags: [Kotlin, Spring, Encoding, UTF-8, EUC-KR, KSC5601]
---

```
Keywords: Spring framework, Kotlin, UTF-8, EUC-KR, KSC5601, KS X 1001, HttpMessageConverter, ObjectMapper, JsonSerialize
```

토스뱅크는 은행으로서 마이데이터 제공자이다. 마이데이터 제공 업무 관련하여 Response DTO의 인코딩에 대해서 알아볼 일이 있었다.

Spring Framework에서 Response DTO를 JSON String & ByteArray로 serialize하는 구조에 대해서 알게 되었다.

- Response body의 encoding을 customize하려면 어떻게 해야할까?
- 서버 전체의 encoding을 변경하는 것은 영향 범위가 너무 크다. Response body의 특정 필드의 encoding만 변경할 수 있을까?

## UTF-8, EUC-KR, KSC5601

해결하고자 하는 문제는, 기존에 UTF-8로 인코딩되던 값을 KSC5601로 인코딩하는 것이었다.

Spring 이야기를 하기 전에, 한글 인코딩 방식들의 차이부터 알아보자.

> 생각해보니 과거에 인코딩 관련하여 정리한 포스트가 있었다: <https://ssaemo.tistory.com/28>
> 
> 당시에는 EUC-KR 인코딩에 대해서 대략적으로만 이해하고 넘어갔었다

### Kotlin/JVM에서 String->ByteArray 변환 구현: String.toByteArray(Charset)

Kotlin/JVM에서는 toByteArray(Charset) extension 함수를 통해 String을 encode 할 수 있다.

> `java.nio.charset.Charset`에는 `java.nio.charset.StandardCharsets`가 포함된다.

```kotlin
"가나다라".toByteArray(Charset.forName("EUC-KR"))
```

인코딩된 ByteArray를 hex 값으로 보고싶다면 아래 함수를 사용할 수 있다.

```kotlin
fun printIt(string: String) {
    println(string)
    for (i in string.indices) {
        print(String.format("U+%04X ", string.codePointAt(i)))
    }
    println()
}
```

그럼 Charset의 name을 KSC5601로 지정하면 되는걸까? 그렇지 않다. KSC5601로 호출 시, EUC-KR Charset이 나온다.

```kotlin
"가나다라".toByteArray(Charset.forName("KSC5601")) // same as EUC-KR
```

### EUC-KR과 다른 인코딩 방식들의 차이

아래 문서들을 참고했다

- KSC5601, EUC-KR, UTF-8 구분: <https://blog.naver.com/amnesty7/30034184321>
- KSC5601, 한글완성형표준: <http://www.ktword.co.kr/test/view/view.php?m_temp1=1452>
- KSC5601 vs EUC-KR vs CP949, KSC5601 vs Unicode: <https://nuli.navercorp.com/community/article/1079940>

#### UTF-8

- 유니코드를 8 bit(1 byte) 단위로 인코딩한 것.
- ASCII code는 1바이트로 처리된다. 한글 1글자를 3바이트로 표현

#### EUC-KR

- EUC-KR = KSC5601(한글) + KSC5636(영문)
- KSC5601로 한글을 표현하기 때문에 모든 한글을 표현하지 못한다.

> EUC: Extended Unix Code

#### KSC5601-87 (KS X 1001)

- 전각문자, 기호, 한글, 한자 등을 표현 가능
- 모든 한글을 표현하지 못함
- 한글 1글자를 2바이트로 표현 (2바이트 완성형)
  - [참고](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=dolicom&logNo=10130052269)
- ASCII code를 포함하지 않는다 (반각 영문자를 표현할 수 없음)

> [나무위키: 완성형](https://namu.wiki/w/%EC%99%84%EC%84%B1%ED%98%95)
> 
> [조합형? 위키피디아에서](https://ko.wikipedia.org/wiki/KS_X_1001): 한글 부분은 기본적으로 2바이트 완성형 코드이지만, 부속서 3에서 2바이트 조합형 코드도 보조 부호계로서 규정되어 있다.
> 
> [한글 조합형 인코딩](https://ko.wikipedia.org/wiki/%ED%95%9C%EA%B8%80_%EC%83%81%EC%9A%A9_%EC%A1%B0%ED%95%A9%ED%98%95_%EC%9D%B8%EC%BD%94%EB%94%A9)
> - MSB(가장 좌측 비트)는 항상 1이다
> - 초성, 중성, 종성을 각 5비트로 변환한다
> - 1 + 초성 (5 bits) + 중성 (5 bits) + 종성 (5 bits) = 16 bits = 2 bytes

특정 한글의 KSC5601 인코딩 hex 값이 궁금하다면 이 페이지를 참조: <https://crazybrain.tistory.com/33>

> KS C 5601은 KS X 1001의 옛 이름이라고 한다: https://ko.wikipedia.org/wiki/KS_X_1001

#### CP949 (Code Page 949)

- EUC-KR의 확장
- 한글 windows 메모장 저장 시 ANSI가 이에 해당

## 한글 인코딩 방식 차이 요약

- 설명이 길었다. 요약
- UTF-8과 EUC-KR(KSC5601)의 한글 인코딩은 바이트 수도 다르고, KSC5601 한글 완성형은 UTF-8 한글 인코딩과 호환되지 않는다.
- EUC-KR은 KSC5601(KS X 1001)을 포함한다.

## Spring framework와 JSON serialization, `ObjectMapper`

fastxml의 Jacson ObjectMapper 클래스가 있다. Spring은 이 클래스를 사용하여 JSON 등 mapping을 구현했다.

JsonSerializer interface를 구현하여 ObjectMapper serialization을 customize할 수 있는데, 기본적으로 String으로 serialize한다.

위와 같이 특수한 경우에는 String이 아닌 ByteArray로 직접 serialize할 수 있다.

```kotlin
class MyStringSerializer : JsonSerializer<String>() {
    override fun serialize(value: String, gen: JsonGenerator, serializers: SerializerProvider) {
        val converted = convertHalfToFullWidthCharacter(value)
        gen.writeString(converted)

        // val bytes = convertToSomething(value)
        // gen.writeUTF8String(bytes, 0, bytes.size)
    }
}
```

### Spring MVC `HttpMessageConverter`

ObjectMapper보다 좀더 상위 수준에서, HTTP response body의 변환은 `HttpMessageConverter` interface에서 담당한다

- <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/message-converters.html>
- <https://docs.spring.io/spring-boot/docs/2.5.14/reference/htmlsingle/#features.developing-web-applications.spring-mvc.message-converters>

#### Jackson의 HttpMessageConverter 구현체

HttpMessageConverter의 구현체가 MappingJackson2HttpMessageConverter이다. 이 클래스의 구현이 궁금하다면 AbstractJackson2HttpMessageConverter.writeInternal() 메소드를 보자.

HttpMessageConverter.write() 메소드 파라미터 중 HttpOutputMessage가 있는데, AbstractGenericHttpMessageConverter.write()의 구현을 보면 HttpOutputMessage.getHeaders()를 통해 HttpHeaders 객체를 얻은 후 addDefaultHeaders() 메소드를 통해 헤더 객체 변경하니 테스트 등에서 HttpOutputMessage 파라미터 구현 시 유의하도록 하자. (default header가 여기에 설정됨. e.g. Content-Type header와 default charset)

- AbstractJackson2HttpMessageConverter.ENCODINGS (com.fasterxml.jackson.core.JsonEncoding enum의 값들을 포함)
- UTF8~UTF32만 존재함. EUC-KR은 존재하지 않음 (EUC-KR과 같이 매칭되는 encoding을 찾지 못할 경우 UTF-8)

## JsonSerialize

Jackson에서 JsonSerialize라는 어노테이션을 제공한다. 이를 통해 특정 필드에 어떤 serializer를 사용할 것인지 지정할 수 있다. (key, value 각각 설정 가능)

```kotlin
class MyResponse(
    @JsonSerialize(using = KSC5601Serializer::class)
    val s: String,
)
```

## Conclusion

허무하지만 나중에 알고보니 오류 원인은 인코딩 문제가 아니었다. 

하지만 과정에서 여러 가지 지식을 배울 수 있었다.

- Kotlin/JVM String->ByteArray 변환 방법
- 다양한 한글 인코딩 방식들의 차이
- Spring Framework에서 encoding, serialization을 처리하는 구조 (`HttpMessageConverter`, `ObjectMapper`의 역할)
- Jackson `@JsonSerialize` 어노테이션

문제 해결 과정에서 ChatGPT의 도움도 받아보았는데, 결과적으로는 ChatGPT가 도움이 된 것 같기도 하고, 잘못된 정보로 방해가 된 것 같기도 하고...
