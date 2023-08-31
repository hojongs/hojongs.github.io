---
layout: post
title: "[TIL] Logback 용어들: Appender, Encoder, Layout"
categories: [Logback]
tags: [TIL, Logging, Logback]
---

Logback XML config를 작성할 때 appender, encoder, layout이라는 용어를 볼 수 있다

이러한 용어들이 무엇인지 간단히 알아보자

## Logback Appender

<https://logback.qos.ch/manual/appenders.html>

> Logback delegates the task of writing a logging event to components called appenders.

쉽게 말하면 Appender는 로그를 작성하는 역할을 한다. `void doAppend(E event);`라는 메소드가 Appender interface의 핵심이다.

로그를 Console, File, Slack, Kafka, Stdout 등 어디에 작성할 것인지는 appender 구현체에 따라 다르다.

또한 로그 작성의 동기/비동기 방식에 따라서도 나뉜다. (AsyncAppender는 내부적으로 synchronous appender를 가지고 이것에 로그 작성을 위임한다)

## Logback Encoder

<https://logback.qos.ch/manual/encoders.html>

> Encoders are responsible for transforming an event into a byte array as well as writing out that byte array into an OutputStream.

Encoder는 logging event 인스턴스를 bytes로 encode(serialize)하는 역할이다

```
(Logging Event) -> Encoder -> (Bytes)
```

사실 encoder 자체를 건드릴 일은 많이 없다. 그저 Appender와 Layout 사이에 있는 객체 정도로 이해하면 된다.

Appender -> Encoder -> Layout

대표적인 구현체는 PatternLayoutEncoder으로, PatternLayout을 layout으로 사용한다. 가장 많이 사용되는 Encoder다.

`void doEncode(E event) throws IOException;` 메소드가 Encoder interface의 핵심이고, bytes로 바꾼 event를 OutputStream에 write한다.

## Layout

<https://logback.qos.ch/manual/layouts.html>

> Layouts are logback components responsible for transforming an incoming event into a String

```
(Logging Event) -> Layout -> (String)
```

Encoder까지 포함하면 이렇게 된다.

```
(Logging Event) -> Layout -> (String) -> Encoder -> (Bytes)
```

`String doLayout(E event);` 메소드가 Layout interface의 핵심이다.

로그를 Console용 String, ES용 JSON 등 어떤 format의 문자열로 출력할 것인지 Layout이 결정한다.

Appender, Encoder, Layout 중 Layout을 설정할 일이 가장 많을 것이다.

PatternLayout이 대표적인 구현체이다.

### Coloring

<https://logback.qos.ch/manual/layouts.html#coloring>

Console 출력 시 글자에 색을 입힐 수 있다. pattern에 아래와 같이 작성하면 된다.

```
%highlight(%-5level) %cyan(%logger{15})
```

## Filter

Filter는 별도로 언급하지 않았는데, 간단하다. level 등으로 필터링하여 일부 로그를 남기지 않고 제외하는 역할이다.

## Logstash Logback Encoder

글 작성하는 김에, JSON format logging을 위한 Logstash의 오픈 소스도 소개한다. 직접 구현하는 것보다 이걸 사용하는 게 더 낫다.

<https://github.com/logfellow/logstash-logback-encoder>

위 라이브러리에서 다양한 Appender, Encoder, Layout들을 제공하는데 이 부분을 보면서 Encoder와 Layout의 차이가 궁금해졌고 그래서 이 포스트로 정리해보았다.

여기서는 간단한 언급만 하고 자세한 사용법은 다음 포스트에서 다룰 예정이다.

위 라이브러리에서 특히 관심있는 기능들은 아래와 같다.

- LogstashLayout (JSON format)
  <https://github.com/logfellow/logstash-logback-encoder#encoders--layouts>
- Graceful Shutdown 기능 등을 제공하는 Async Appenders
  <https://github.com/logfellow/logstash-logback-encoder#async-appenders>
- Customizing Stack Traces
  <https://github.com/logfellow/logstash-logback-encoder#customizing-stack-traces>

### Q: Encoder와 Layout 중 어떤 것을 사용해야 할까?

Encoder는 로그를 남기는 매체에 따라 결정된다. 

로그를 파일에 남길 경우 LogstashEncoder를 사용할 수 있다. (README에 나와있듯이)

LogstashEncoder는 아마 OutputStream에 JSON을 그대로 write하는 것 같다. (기본값)

prefix, suffix 등을 통해 쉽게 customize할 수 있다.

하지만 로그를 Kafka로 보낼 경우 Kafka Encoder를 사용해야 한다. 로그를 Slack으로 보낼 수도 있는데, 이 경우에는 Slack encoder를 사용해야 하는 것이다.

> README를 읽어보니, LogstashUdpSocketAppender는 encoder가 아니라 layout을 설정해야 한다. 이는 appender 구현체에 완전히 의존적이다. 확실한 것은 Logback이 0.9.19 버전부터 Layout으로부터 Encoder가 분리되었다는 것이다.

반면 Layout은 로그를 남기는 위치와 독립적이다.

따라서 LogstashEncoder는 로그를 남기는 위치에 따라 적절히 customize하거나 다른 encoder를 사용할 수 있다.

**Encoder와 달리 LogstashLayout는 JSON logging을 위해 필수다.**

본인의 경우 기존에 사용하던 Kafka encoder에 layout만 변경하여 사용할 계획이다.

## Conclusion

Logback 관련 내용들을 간단하게 알아보았다. 다음 Logback 포스트는 Logstash Logback Encoder가 될 예정이다.
