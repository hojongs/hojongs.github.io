---
layout: post
title: "Spring Boot + Elasticsearch 사용법 및 동작방식"
categories: [Server development]
tags: [Spring Data, Elasticsearch, Spring Boot]
---

Spring ecosystem에는 대부분의 외부 시스템을 위한 모듈이 있다. Elasticsearch도 마찬가지다. Elasticsearch를 위한 Spring Boot와 Spring Data에 대하여 알아본다.  
추가로, 본인이 회사 업무에서 마주했던 use case들도 작성해보았다.

## Spring boot starter data elasticsearch

Spring 애플리케이션에 Elasticsearch client를 가장 간단하게 구현하는 방법은 Spring Boot Starter Data Elasticsearch를 사용하는 것이다.

- Docs: <https://docs.spring.io/spring-boot/docs/3.0.2/reference/html/data.html#data.nosql.elasticsearch>
- Source code: <https://github.com/spring-projects/spring-boot/tree/v3.0.2/spring-boot-project/spring-boot-starters/spring-boot-starter-data-elasticsearch>

공식 문서에서 언급된대로, 사용 가능한 Elasticsearch client는 다양한 종류가 있다.

- low-level REST client (Official)   
  `org.elasticsearch.client:elasticsearch-rest-client`
- Java API client (Official)  
  `co.elastic.clients:elasticsearch-java`
- ReactiveElasticsearchClient (Spring data elasticsearch)  
  `org.springframework.data:spring-data-elasticsearch`

### Usage

의존성을 추가한다.

```kotlin
dependencies {
    // https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-elasticsearch
    implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch")
}
```

아래 property들을 설정하면 된다.

```properties
spring.elasticsearch.uris=https://search.example.com:9200
spring.elasticsearch.socket-timeout=10s
spring.elasticsearch.username=user
spring.elasticsearch.password=secret
```

#### Customize RestClient

`elasticsearch-rest-client` 의존성이 classpath에 존재하면, RestClient bean이 auto-configure 된다. RestClient bean을 설정하려면 `RestClientBuilderCustomizer` bean을 등록하면 된다.

```kotlin
@Configuration
class ElasticsearchConfig {
    fun restClientBuilderCustomizer(): RestClientBuilderCustomizer {
        // ...
    }
}
```

좀더 세밀한 설정이 필요하다면 `RestClientBuilder` bean을 등록하면 된다.

```kotlin
@Configuration
class ElasticsearchConfig {
    fun restClientBuilder(): RestClientBuilder {
        // ...
    }
}
```

##### RestClientBuilderCustomizer vs RestClientBuilder

개인적으로, `RestClientBuilderCustomizer`는 추상화가 실패했다고 생각한다. 사용해보면서 결국 내부 구현을 이해해야 했기 때문이다.
RestClientBuilderCustomizer bean을 사용 시 경우에 따라, spring.elasticsearch.* property들이 적용되지 않는다는 사실을 알아야 했다.

**RestClient Auto configuration 구현과 RestClientBuilderCustomizer 사용 시 주의할 점:**

- DefaultRestClientBuilderCustomizer bean은 항상 등록된다. 그리고 이 customizer가 spring.elasticsearch.* property들을 RestClient에 적용한다.
- DefaultRestClientBuilderCustomizer는 customize(HttpAsyncClientBuilder builder), customize(RequestConfig.Builder builder) 메소드를 override해서 property들을 RestClient에 적용한다.
- 이 customize() 메소드들은 아래 링크의 auto configuration 파일들에서 호출된다.
- 만약 우리가 정의한 bean의 `customize(builder: RestClientBuilder)` 메소드에서 **`RestClientBuilder.setHttpClientConfigCallback()` 메소드를 호출할 경우**, DefaultRestClientBuilderCustomizer 등이 적용된 callback은 replace 당한다.
- `RestClientBuilder.setRequestConfigCallback()` 메소드도 마찬가지다. customizer에서 호출 시 다른 customizer들이 적용된 callback을 replace 해버린다. (의도치 않게)
- 다른 customizer들의 변경사항과 함께 적용되려면, `customize(HttpAsyncClientBuilder builder)`와 같은 메소드를 구현해야 한다. 하지만 이는 충분히 놓치기 쉽다.
- 위 방식으로 구현한다고 해도, customizer들이 적용되는 순서도 영향이 있을텐데 이 순서는 예측하기가 어렵다. (만약 `@Order` 어노테이션을 붙이지 않았을 경우) 심지어 DefaultRestClientBuilderCustomizer bean도 order가 명시되지 않았기 때문이다. (아마 LOWEST_PRECEDENCE로 취급되겠지만)

결론적으로 Customizer의 동작이 예측하기 어렵다고 생각하기 때문에, 본인은 Customizer로 시도하다가 RestClientBuilder bean을 정의하는 것으로 리팩토링하였다.

자세한 구현은 아래 파일을 참조하면 된다.  
Spring boot 2.7: <https://github.com/spring-projects/spring-boot/blob/v2.7.8/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/elasticsearch/ElasticsearchRestClientConfigurations.java#L76-L87>
Spring boot 2.4: <https://github.com/spring-projects/spring-boot/blob/v2.4.13/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/elasticsearch/ElasticsearchRestClientAutoConfiguration.java#L71-L79>

> 구현은 거의 동일하지만 파일 이름과 위치가 달라서 별도로 링크를 남겼다.

#### Auto-configure Sniffer

`elasticsearch-rest-client-sniffer` 의존성이 classpath에 존재하면, Sniffer bean이 auto-configure 된다. Sniffer는 자동으로 ES cluster에서 node들을 discover하기 위해 사용되며, RestClient에 설정된다. 아래 property들은 Sniffer를 설정한다.

```properties
spring.elasticsearch.restclient.sniffer.interval=10m
spring.elasticsearch.restclient.sniffer.delay-after-failure=30s
```

#### Auto-configure Elasticsearch client

ElasticsearchClient bean도 마찬가지로, `co.elastic.clients:elasticsearch-java` 의존성이 classpath에 존재하면 auto-configure 된다.
ElasticsearchClient bean은 내부적으로 위에서 설명한 RestClient bean을 사용한다.

더 나아가 `TransportOptions` bean을 통해 추가 설정이 가능하다.


```kotlin
@Configuration
class ElasticsearchConfig {
    fun transportOptions(): TransportOptions {
        // ...
    }
}
```

### Elasticsearch with Spring Data

ElasticsearchClient bean이 등록되어 있으면 `ElasticsearchTemplate` bean도 auto-configure 된다.
`ElasticsearchTemplate`의 reactive 버전으로서, `spring-data-elasticsearch`과 `reactor`가 classpath에 존재할 경우 `ReactiveElasticsearchClient`과 `ReactiveElasticsearchTemplate` 클래스가 bean으로 auto-configure 된다.

### Elasticsearch with Spring Data Repository

`@Document` 어노테이션이 붙은 클래스가 존재한다면, 해당 엔티티에 대한 Spring Data Repository가 auto-configure 된다. 이 repository를 사용하지 않을 경우, 아래 property를 통해 auto-configure 되지 않도록 할 수 있다. (사실 repository를 사용하지 않으면 `@Document` 어노테이션을 사용할 일도 없을 것 같다.)

```properties
spring.data.elasticsearch.repositories.enabled=false
```

이제, Spring Data Elasticsearch 문서를 살펴볼 차례다.  
<https://docs.spring.io/spring-data/elasticsearch/docs/5.0.1/reference/html/>

> 보통은 Spring data elasticsearch 4.x를 사용하고 있을 것이다. 아래 내용은 5.0을 기준으로 작성하였으니, 유의해야 한다.

## Spring Data Elasticsearch

### Spring Data Elasticsearch version compatibility

Spring Data Elasticsearch를 사용할 때는 Elasticsearch server 버전 사이 호환성을 잘 확인해야 한다.  
<https://docs.spring.io/spring-data/elasticsearch/docs/5.0.1/reference/html/#preface.versions>

### ClientConfiguration bean

Spring boot에서는 property들로 설정하지만, Spring data elasticsearch에서는 RestClient를 위해 ClientConfiguration bean을 등록해야 한다.  
<https://docs.spring.io/spring-data/elasticsearch/docs/5.0.1/reference/html/#elasticsearch.clients.restclient>

그러면 3개의 bean이 auto-configure 된다: ElasticsearchOperations, ElasticsearchClient, RestClient.
이 ClientConfiguration bean은 reactive bean들을 설정할 때도 공통적으로 쓰인다.

### Spring data elasticsearch 5.x vs 4.x

하지만 이건 Spring Data Elasticsearch 5.0부터 변경된 사항이고, 4.4.7 문서를 보면 사용법이 다르다.  
<https://docs.spring.io/spring-data/elasticsearch/docs/4.4.7/reference/html/#reference>

4.4.7에서는 ClientConfiguration bean 대신 RestHighLevelClient, ReactiveElasticsearchClient bean을 등록하도록 가이드 되어있다.

> Elasticsearch Java High-level REST client는 deprecated 되었디.
> https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.17/java-rest-high.html

### Spring boot 버전과 Spring data elasticsearch 버전 관계

막상 5.0을 기준으로 내용을 작성하고 보니, 현재 production에서 사용되고 있는 버전은 대부분 4.x일 것이라는 생각이 들었다. 그 이유를 조금 살펴보려 한다.

Spring data elasticsearch 5.x는 Spring boot 3.0부터 사용한다.
Spring boot 2.7.8에서는 Spring data elasticsearch 4.4.7을 사용하고 있다.

Spring boot 3.0은 Java 17이 최소 요구사항이기 때문에, production에서 사용하기에는 꽤나 geek하다. (risky하다)  
<https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Release-Notes>

### `@Document`: createIndex=true (default)

`@Document` 어노테이션이 붙은 클래스가 존재할 경우, createIndex=true임에 주의해야 한다.  
<https://docs.spring.io/spring-data/elasticsearch/docs/5.0.1/reference/html/#elasticsearch.repositories.autocreation>

Spring 애플리케이션이 실행될 때마다 자동으로 document 이름의 ES index가 생성될 수 있다.

## Use cases

본인이 현재 회사의 업무동안 Spring Data Elasticsearch를 다루면서 마주했던 사례들을 추가로 다뤄보려 한다.

### Spring Data Elasticsearch with Multi-clusters

일반적으로 Elasticsearch client는 하나만 요구한다. Single ES cluster에 연결하기 때문이다.

본인이 작업했던 서버는 주소(address) 서버였는데, 2개의 ES cluster에 연결하는 서버 사례였다. 회사가 고가용성을 제공하기 위해 2개의 데이터 센터를 운영하고 있었고, ES cluster 역시 2개의 데이터 센터에 각각 배포되어 있었기 때문이다.  

이런 경우에는 당연히 property를 통해 여러 개의 RestClient를 설정할 수 없다. spring.elasticsearch.* properties는 단일 클라이언트를 위한 설정이기 때문이다. 따라서 2개의 client를 직접 build하는 코드를 작성해야겠고, 두 client는 Elasticsearch uri도 당연히 달랐다.

> client 설정 시 uri나 host 여러 개를 파라미터로 받는다. 이는 client가 요청을 보낼 단일 cluster의 node들의 uri 혹은 host이다. Multi-cluster에 동일 요청을 보내야하는 이 use case에는 해당되지 않는다.

여담으로, 각 client build 시 설정이 적용된 RestClientBuilder를 정의하고 재사용할 수가 없다는 번거로움이 있었다. RestClient builder 인터페이스 상 builder 인스턴스 생성 후 uri를 변경할 수 없었기 떄문이다. (builder 생성자에서 uri를 파라미터로 받는다)  

### Elasticsearch 트래픽 전환과 connection TTL (Active-active)

회사 DevOps 팀에서 DC(Data center) 자체를 Active-active로 구성해놓았기 때문에 우리는 언제든지 ES cluster 1에서 cluster 2로 트래픽을 전환할 수 있다.
Active-active infra architecture 구성에 대해서는 아래 영상에서 잘 설명해주고 있다.  

<iframe width="560" height="315" src="https://www.youtube.com/embed/ftFHZwyUN38?start=173" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

이 떄 트래픽 전환은 도메인의 IP 변경을 통해서 수행하고 있다. 이 때 문제점은 도메인의 변경된 IP로 ES 커넥션을 다시 맺어야한다는 것이다.

ES client는 HTTP protocol을 사용하고, HTTP/1.1 protocol은 기본적으로 일정 시간동안 connection을 유지한다. 즉, 커넥션이 계속 유지될 수 있다.  
이 이슈를 해결하기 위해 커넥션을 요청 시마다 맺도록 하는 것도 하나의 방법일 수 있겠지만 성능 하락을 감안해야 했다. 다행히도 더 좋은 방법이 있었다.

RestClient 내부적으로 사용하는 Apache HttpAsyncClient에서 connection TTL 설정을 제공했다. 처음에는 이 설정이 없다고 생각했는데, Spring boot 2.4에서는 Apache HttpAsyncClient 의존성 4.1.4 버전을 사용하고 해당 파라미터는 4.1.5에서 추가되었기 때문에 발생한 오해였다. (팀원 분이 해당 설정을 찾아주셔서 알게 되었다)

<https://downloads.apache.org/httpcomponents/httpasyncclient/RELEASE_NOTES-4.1.x.txt>

> Release 4.1.5
> -------------------
> 
> This is a maintenance release that fixes a number of issues discovered since 4.1.4.
> 
> Changelog
> -------------------
>
> ...
> Added connection time to live parameter to HttpAsyncClientBuilder.

따라서 해당 의존성 버전을 4.1.5로 올리고, connection TTL 설정을 추가하여 ES 도메인 IP 변경 후 커넥션이 다시 연결될 수 있도록 할 수 있었다.

이와 관련하여 좀더 별도로 포스트를 작성하였다: [애플리케이션 서버 HTTP Client에 도메인 IP 변경 전파하는 방법](https://hojongs.github.io/posts/how-to-propagate-elasticsearch-domain-ip-change-to-application-http-client/)

## Conclusion

Elasticsearch 관련 Spring boot, Spring data 사용법과 몇몇 트러블슈팅 사례를 공유했다. 추가적으로 실제 업무에서 Spring data elasticsearch 관련 약간 더 깊은 수준의 사례들을 공유해보았다.
