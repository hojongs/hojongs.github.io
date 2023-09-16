---
layout: post
title: "[TIL] Logstash Logback Encoder 사용방법"
categories: [Logback]
tags: [TIL, Logging, Logback]
---

<https://github.com/logfellow/logstash-logback-encoder#customizing-standard-field-names>

<https://github.com/logfellow/logstash-logback-encoder#event-specific-custom-fields>

- message 필드가 중복으로 들어감
- standard field인 message를 제거 (`[ignore]`)


StructuredArguments
Markers

<https://github.com/logfellow/logstash-logback-encoder#context-fields>

> By default, each property of Logback's Context (ch.qos.logback.core.Context) will appear as a field in the LoggingEvent. This can be disabled by specifying <includeContext>false</includeContext> in the encoder/layout/appender configuration.
