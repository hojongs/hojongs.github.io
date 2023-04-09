---
layout: post
title: "Slack Bolt: Socket Mode App Server를 Spring Boot로 통합하기"
categories: [Server development]
tags: [Slack Bolt, Spring Boot]
redirect_from:
  - /posts/integrate-slack-bolt-socket-mode-app-server-into-spring-boot/
permalink: /integrate-slack-bolt-socket-mode-app-server-into-spring-boot/
---

## Slack app server를 개발하고 싶다. 안전하게.

기존에 개발되어 있는 Slack app server는 Slack hook endpoint를 노출한 형태로 개발되어 있었다.

하지만 이 형태는 Public endpoint를 노출할 수 밖에 없다.

signing secret으로 인증을 하긴 하지만 API 호출 자체는 누구나 할 수 있다.

더 편리하게, 그리고 **Public HTTP endpoint 노출 없이** Slack app server를 개발해보자.

## Slack Bolt?

Slack은 더 편리한 Slack app server 개발을 위해 **Slack Bolt**라는 것을 출시하였다.

또한 Slack Bolt에서는 **Socket mode**라는 기능을 제공한다. Websocket server 형태로 Public endpoint 노출 없이 Slack app server를 운영할 수 있음을 의미한다.

이 포스트에서는 Slack Bolt를 사용하여 Slack app server를 개발하는 방법을 간단히 소개한다.

### With Spring boot

기존 Spring boot 서버에서 함께 serving 하기 위해 그리고 Spring boot에 구현된 여러 가지 기능들을 통합하기 위해 **Slack bolt를 Spring boot 속으로 통합**하는 방법까지 함께 소개한다.

## Getting started with Bolt

> Slack app 생성, 설정, 워크스페이스에 설치는 되어있다고 가정한다.

공식 문서 예제부터 살펴보자

<https://slack.dev/java-slack-sdk/guides/getting-started-with-bolt>

```kotlin
dependencies {
  implementation("com.slack.api:bolt-jetty:1.28.1")
  implementation("org.slf4j:slf4j-simple:1.7.36")
}
```

```java
package hello;

import com.slack.api.bolt.App;
// If you use bolt-jakarta-jetty, you can import `com.slack.api.bolt.jakarta_jetty.SlackAppServer` instead
import com.slack.api.bolt.jetty.SlackAppServer;

public class MyApp {
  public static void main(String[] args) throws Exception {
    // App expects env variables (SLACK_BOT_TOKEN, SLACK_SIGNING_SECRET)
    App app = new App();

    app.command("/hello", (req, ctx) -> {
      return ctx.ack(":wave: Hello!");
    });

    SlackAppServer server = new SlackAppServer(app);
    server.start(); // http://localhost:3000/slack/events
  }
}
```

예제에서는 Jetty를 사용하여 서버를 실행한다.

### Slack Bolt with Spring Boot

Slack Bolt를 Spring Boot와 integrate하는 방법도 소개하고 있다. 하지만 socket mode와 관련해서는 문서가 없다.

<https://slack.dev/java-slack-sdk/guides/supported-web-frameworks>

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'com.squareup.okhttp3:okhttp:4.10.0'
  implementation 'com.slack.api:bolt-servlet:1.28.1'
}
```

```java
package hello;

import com.slack.api.bolt.App;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SlackApp {
  @Bean
  public App initSlackApp() {
    App app = new App();
    app.command("/hello", (req, ctx) -> {
      return ctx.ack("What's up?");
    });
    return app;
  }
}

@WebServlet("/slack/events")
public class SlackAppController extends SlackAppServlet {
  public SlackAppController(App app) {
    super(app);
  }
}
```

## Getting started with Slack Bolt socket mode

<https://slack.dev/java-slack-sdk/guides/getting-started-with-bolt-socket-mode>

![slack-app-enable-socket-mode](/assets/img/posts/slack-app-enable-socket-mode.png)

```kotlin
dependencies {
  implementation("com.slack.api:bolt-socket-mode:1.28.1")
  implementation("javax.websocket:javax.websocket-api:1.1")
  implementation("org.glassfish.tyrus.bundles:tyrus-standalone-client:1.19")
  implementation("org.slf4j:slf4j-simple:1.7.36")
}
```

```java
package hello;

import com.slack.api.bolt.App;
import com.slack.api.bolt.socket_mode.SocketModeApp;

public class MyApp {
  public static void main(String[] args) throws Exception {
    // App expects an env variable: SLACK_BOT_TOKEN
    App app = new App();

    app.command("/hello", (req, ctx) -> {
      return ctx.ack(":wave: Hello!");
    });

    // SocketModeApp expects an env variable: SLACK_APP_TOKEN
    new SocketModeApp(app).start();
  }
}
```

Slack Bolt 예제에서 생성한 App 인스턴스를 wrapping한 SocketModeApp 인스턴스를 생성하여 start() 메소드를 호출하면 된다.

하지만 start() method는 thread를 block한다. 대신 startAsync() 메소드가 존재한다.

## Slack Bolt socket mode with Spring Boot

우리의 목표는 Spring Boot 서버가 시작됐을 때 Slack Bolt 서버를 init 하고 startAsync() 메소드를 호출하면 되는 것이다. 이를 아래와 같이 구현하였다.

- Spring Boot 서버가 시작됐을 때: ContextStartedEvent가 발생했을 때
- Slack Bolt server 초기화: event listener에서 초기화 및 startAsync() 메소드 실행

### Spring framework standard events

<https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-functionality-events>

여러 가지 standard event 중 ContextStartedEvent는 ApplicationContext(IoC container)가 start 되었을 때 publish된다.

event listener를 구현해보자.

<https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-functionality-events-annotation>

```kotlin

private val logger = KotlinLogging.logger {}

@Configuration
class SlackBoltServerConfig {
    @Bean
    @EventListener(ContextStartedEvent::class)
    fun startSocketModeApp() {
        logger.info("Start socket mode slack bolt app server.")
    }
}
```

NOTICE: EventListener 어노테이션과 함께 Bean 어노테이션을 사용하여, 반드시 bean으로 등록해줘야 호출된다.

> logging은 kotlin-logging 라이브러리이다. println()으로 대체해도 상관없다.

위 Configuration이 ComponentScan 되도록 배치하고 Spring boot 서버를 실행해보면 위 log가 출력되는 것을 확인할 수 있을 것이다.

```kotlin
private val logger = KotlinLogging.logger {}

@Configuration
class SlackBoltServerConfig {
    @Bean
    @EventListener(ContextStartedEvent::class)
    fun startSocketModeApp() {
        logger.info("Start socket mode slack bolt app server.")
        val botToken = System.getenv("SLACK_BOT_TOKEN")
        logger.info("botToken: $botToken")

        val boltApp = App().apply {
            command("/hello") { req, ctx -> ctx.ack("What's up?") }
            event(AppMentionEvent::class.java) { event, context: EventContext ->
                logger.info("event=$event, context=$context")
                Response.ok()
            }
        }
        SocketModeApp(boltApp).startAsync()
    }
}
```

위 코드는 `/hello` 커맨드를 입력했을 때와 app mention event가 발생했을 때, 각각 커맨드와 이벤트를 받아서 핸들링하는 코드다.

<https://api.slack.com/events/app_mention>

> NOTE: 다른 이벤트를 수신하려면 해당하는 scope을 slack app에 추가해야 한다.

### Slack Tokens: Bot Token, App Token

두 개의 토큰이 필요하다. SLACK_BOT_TOKEN은 `xoxb-`로 시작하는 토큰이고 SLACK_APP_TOKEN은 `xapp-`으로 시작하는 토큰이다.

두 토큰을 발급받도록 하자. 아래 문서를 참고하면 된다.

<https://slack.dev/java-slack-sdk/guides/getting-started-with-bolt-socket-mode>

SLACK_APP_TOKEN은 connections:write scope이 설정되어 있어야 한다.

SLACK_BOT_TOKEN의 scope은 `app_mentions:read`같은 것을 추가해주면 된다.

---

안타깝게도 Slack Bolt 앱은 반드시 "환경 변수"로만 토큰 설정을 받는다.

bootRun 실행 시 Spring boot 서버에 환경 변수를 설정하기 위해 아래와 같이 

```kotlin
tasks {
    withType<BootRun> {
        environment(
            "SLACK_BOT_TOKEN",
            "xoxb-****",
        )
        environment(
            "SLACK_APP_TOKEN",
            "xapp-****",
        )
    }
}
```

추가한 로그를 통해 환경 변수가 잘 주입되었는지 확인하도록 하자.

보안을 위해 build script에 token을 넣지 않으려면 아래와 같이 하자. home의 gradle properties에 token을 저장하자.

```properties
# $HOME/.gradle/gradle.properties
slack.token.bot=xoxb-****
slack.token.app=xapp-****
```

```kotlin
tasks {
    withType<BootRun> {
        findProperty("slack.token.bot")?.let { environment("SLACK_BOT_TOKEN", it) }
        findProperty("slack.token.app")?.let { environment("SLACK_APP_TOKEN", it) }
    }
}
```

만약 위 property들이 없을 경우 **Spring app 실행이 실패하므로,** slack app server 실행 메소드 전체를 runCatching {} 으로 묶어서 Slack app server 실행이 실패해도 Spring app 실행은 진행되도록 수정하였다.

### Conclusion

여차저차하여 Slack token까지 설정하고 위 예제 코드로 Spring boot 서버를 실행하면 Spring boot 서버와 함께 Slack Bolt socket mode 서버를 실행할 수 있다.

서버를 실행 후 Slack app이 설치된 workspace에서 해당 Slack app을 멘션해보면 이벤트를 수신하는 것을 확인할 수 있다.

이로써 public endpoint 없이 Slack app server를 serving할 수 있게 되었다.

public endpoint가 없다는 것은 해당 이벤트를 여러 서버에서 수신할 수 있다는 것을 의미하고, 이것은 Slack app server를 개발할 때에도 매우 편리하다. (ngrok을 사용하지 않아도 된다. 이 또한 보안 측면에서 좋은 점이다.)

## References

- <https://slack.dev/java-slack-sdk/guides/getting-started-with-bolt>
- <https://slack.dev/java-slack-sdk/guides/getting-started-with-bolt-socket-mode >
- <https://slack.dev/java-slack-sdk/guides/supported-web-frameworks>
