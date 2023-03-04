---
layout: post
title: "애플리케이션 서버 HTTP Client에 도메인 IP 변경 전파하는 방법 (Elasticsearch)"
categories: [Server development]
tags: [Spring Boot, Spring Data, Spring Data Elasticsearch, Elasticsearch, HTTP, Network, Network TTL, DNS, GSLB]
---

# 갑자기 Elasticsearch 도메인의 IP가 바뀐다면?

> 어떤 서비스가 `es.example.com` 도메인을 통해 Elasticsearch에 연결하고 있었다고 하자. 최초 연결 시에는 도메인에 a.b.c.d IP가 등록되어 있었는데 갑자기 x.y.z.q IP로 변경된다면?

우리 팀에서 운영하는 여러가지 마이크로서비스 중 주소검색 서비스가 있고 **백엔드 저장소로 Elasticsearch를 사용**하고 있다.

그리고 이 주소 Elasticsearch 저장소는 **Active-active 구조로 이중화**되어 있다. 인프라 작업 또는 장애 영향으로 한 쪽의 active Elasticsearch가 전체 트래픽을 처리하기도 한다. 이렇게 트래픽을 전환하기 위해 GSLB 장비에서 도메인을 관리하고 **Elasticsearch 도메인의 IP를 변경**한다.

이 때 문제가 있었는데, **변경된 도메인 IP 정보가 서비스에 전파되지 않았다.** 서비스 재배포를 통해 해결할 수는 있었지만 번거롭고 놓치기 쉬운 부분이었다.

이를 해결하기 위해 ES(Elasticsearch) Connection에 TTL을 적용했다.

# 잠깐, HTTP는 Connectionless인데?

ES Connection이라니 무슨 말인가? 서비스는 ES와 HTTP로 통신하는데? HTTP는 Connectionless인데?

그건 `HTTP/1.0`에 해당하는 것이고, Keep-alive라고 널리 알려진 기능을 통해 **HTTP 커넥션을 유지할 수 있다.** `HTTP/1.1`에서는 persistent connection이라는 기능이 추가되어 일반적으로 서버(여기서는 ES)에 설정된 시간동안 TCP 커넥션이 유지된다.

예전에 작성한 아래 포스트에서 간단히 언급한 적이 있다.

[https://medium.com/jongho-developer/why-use-http-2-over-http-8d30a749eb6d](https://medium.com/jongho-developer/why-use-http-2-over-http-8d30a749eb6d)

> 그 후 공식적으로 HTTP/1.1에서는 이를 개선하기 위해 persistent connection 기능이 추가됨. 즉, 기본적으로 커넥션이 영구적인 것으로 간주함.

# 그래서 Elasticsearch 도메인 IP 변경 전파 방법은?

가장 간단한 옵션 2가지가 있었다.

1. ES 요청 시마다 connection을 새로 맺는다. [Connection HTTP header](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Connection)를 사용하면 된다.
    ```
    Connection: close
    ```
    
2. **ES HTTP Client에 connection TTL을 설정한다.**
    ES Client 라이브러리는 내부적으로 [Apache HttpAsyncClient](https://hc.apache.org/httpcomponents-asyncclient-4.1.x/index.html)를 사용하고 있었다.
    
    다행히도 HttpAsyncClient에서 connection TTL 기능을 지원하고 있었다. (`httpasyncclient:4.1.5`부터)
    
    [https://downloads.apache.org/httpcomponents/httpasyncclient/RELEASE_NOTES-4.1.x.txt](https://downloads.apache.org/httpcomponents/httpasyncclient/RELEASE_NOTES-4.1.x.txt)
    
    ```
    Release 4.1.5
    -------------------
    
    ...
    
    * Added connection time to live parameter to HttpAsyncClientBuilder.
    ```
    
    builder를 customize해서 connection ttl을 설정해주면 된다.

1번 방법은 Elasticsearch query 요청 시마다 새로운 TCP connection을 맺는 오버헤드가 컸다. 이는 Elasticsearch에 불필요한 부하를 발생시키고 서비스 response latency를 증가시키는 단점이 있다. 그래서 **2번 방법을 선택했다.**

# httpasyncclient >= 4.1.5

만약 httpasyncclient의 버전이 4.1.5 미만이라면 Gradle에서 버전업이 필요하다.

```kotlin
dependencies {
    implementation("org.apache.httpcomponents:httpasyncclient:4.1.5")
}
```

# Elasticsearch Client를 customize하는 방법

```kotlin
val client: RestHighLevelClient = RestHighLevelClient(
    RestClient.builder(HttpHost.create(host))
        .setHttpClientConfigCallback { httpClientBuilder: HttpAsyncClientBuilder ->
            httpClientBuilder
                .setDefaultCredentialsProvider(getCredentialsProvider(user, password))
                .setConnectionTimeToLive(30, TimeUnit.SECONDS) // HERE!
        }
)
```

설명: Java 라이브러리인 elasticsearch-client의 RestHighLevelClient는 내부적으로 RestClient를 사용한다. 또 이것은 내부적으로 HttpAsyncClient를 사용한다.

HttpClientConfigCallback을 설정하여 HttpAsyncClientBuilder를 설정할 수 있다.

## Spring Boot Starter Data Elasticsearch에서 customize하는 방법

Spring Boot 2.4 기준으로 Connection TTL을 설정해주는 property는 없다. 따라서 직접 customize해야 한다.

RestClientBuilderCustomizer 또는 RestClientBuilder를 정의하는 2가지 방법이 있다. [이전 포스트]에서 다룬 것처럼 **RestClientBuilder bean을 정의하는 방법을 추천한다.** 자세한 설명은 이전 포스트 참고.

[이전 포스트]: https://hojongs.github.io/posts/spring-boot-elasticsearch-usage-and-mechanism/#restclientbuildercustomizer-vs-restclientbuilder

RestClientBuilder bean을 직접 정의하면 ElasticsearchRestClientProperties의 property들이 적용되지 않는다. 따라서 직접 DI 받아서 RestClientBuilder bean에 설정해줘야 한다.

```kotlin
@Configuration
class ElasticsearchConfig(
    @Value("\${myconfig.elasticsearch.connection-ttl-seconds}")
    private val connectionTtlSeconds: Long,
    private val properties: ElasticsearchRestClientProperties,
) {
    @Bean
    fun restClientBuilder(): RestClientBuilder {
        val hosts = properties.uris.map { HttpHost.create(it) }.toTypedArray()
        return RestClient.builder(*hosts)
            .setHttpClientConfigCallback { httpClientBuilder: HttpAsyncClientBuilder ->
                httpClientBuilder
                    .setDefaultCredentialsProvider(SimpleCredentialsProvider(properties))
                    .setConnectionTimeToLive(connectionTtlSeconds, TimeUnit.SECONDS)
            }
            .setRequestConfigCallback { requestConfigBuilder: RequestConfig.Builder ->
                requestConfigBuilder
                    .setConnectTimeout(properties.connectionTimeout.toMillis().toInt())
                    .setSocketTimeout(properties.readTimeout.toMillis().toInt())
            }
    }

    private class SimpleCredentialsProvider(
        properties: ElasticsearchRestClientProperties,
    ) : BasicCredentialsProvider() {
        init {
            setCredentials(
                AuthScope.ANY,
                UsernamePasswordCredentials(properties.username, properties.password)
            )
        }
    }
}
```

```properties
spring.elasticsearch.rest.uris=https://es.example.com
spring.elasticsearch.rest.username=myuser
spring.elasticsearch.rest.password=mypassword
spring.elasticsearch.rest.connection-timeout=5s
spring.elasticsearch.rest.read-timeout=10s
myconfig.elasticsearch.connection-ttl-seconds=30
```

# Conclusion

위와 같이 적용 후 Elasticsearch Connection이 30초마다 새로 생성되는 것을 확인할 수 있었다. 커넥션 생성 시마다 DNS resolustion이 발생하여 변경된 도메인 IP로 resolve되었다.

따라서 Elasticsearch Active-active 이중화 구조에서 애플리케이션(주소검색 서비스) 재배포 없이 Elasticsearch 트래픽을 전환할 수 있었다.

# References

- [Spring Boot Elasticsearch (2.4.13)](https://docs.spring.io/spring-boot/docs/2.4.13/reference/html/spring-boot-features.html#boot-features-elasticsearch)
- [Spring Boot Elasticsearch (Latest)](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.nosql.elasticsearch)
- [Spring Data Elasticsearch](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#reference)
- [Spring Boot + Elasticsearch 사용법 및 동작방식](https://hojongs.github.io/posts/spring-boot-elasticsearch-usage-and-mechanism/#restclientbuildercustomizer-vs-restclientbuilder)
- 본문 내 링크들
