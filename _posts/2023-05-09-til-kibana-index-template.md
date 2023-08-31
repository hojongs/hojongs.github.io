---
layout: post
title: "[TIL] Elasticsearch: Index template과 dynamic/explicit mapping"
categories: [Elasticsearch]
tags: [Elasticsearch, Kibana, DevOps, TIL, Index template, Elasticsearch mapping]
---

## Problem to resolve

로그를 Elasticsearch에 적재하고 있고, 로그 인덱스는 dynamic mapping을 사용하고 있다

로그 인덱스에서 `message.response.body`라는 필드가 부분 텍스트 검색이 되지 않는 문제를 발견했다.

## Background

Dynamic mapping에 대해서 간단히 살펴보자.

인덱스의 mapping을 조회해보면 top-level에서 `dynamic = true`로 설정된 것을 볼 수 있다 (기본값이 true)

<https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic.html>

dynamic mapping이 이점도 있지만 가끔 다른 형태의 값이 들어오면 잘못된 타입으로 매핑되는 등의 이슈가 있다. 예를 들면 datetime 타입의 필드의 타입이 갑자기 바뀌는 문제가 있었다.

이번 케이스는 `message.response.body` 필드의 타입이 keyword로 매핑된 것이 원인이었다. 이 필드의 타입을 text로 바꿔주면 된다.

## How to solve

Elasticsearch에는 index template이라는 것이 존재한다. index template은 특정 패턴(e.g. log-* pattern)의 index가 생성되면 해당 인덱스의 mapping 등 기본 설정을 제공해준다.

<https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html>

index template의 mapping을 수정하면 특정 필드들의 타입을 명시적으로 매핑해주도록 할 수 있다. mappings의 properties 필드에 값을 추가하면 된다

```json
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
```

<https://www.elastic.co/guide/en/elasticsearch/reference/current/explicit-mapping.html>

Kibana UI에서는 아래와 같다. Kibana admin에 접속해야 한다.

Stack management -> index management -> index templates

template 검색 후 template 선택 -> manage -> Edit

![kibana-index-template](/assets/img/posts/kibana-index-template.png)

Edit 페이지 접근 후 `Step 4: Mapping`를 클릭한다.

그럼 Mapped fields 탭에서 `Add field` 버튼을 통해 필드 이름과 타입을 지정하여 필드를 추가할 수 있다. 필드 추가를 완료하고 index template을 저장하면 된다.

### Tip: multi-field property

multi-field: 추가된 필드 옆에는 `Add a multi-field` 버튼이 노출된다. 이 버튼을 눌러서 multi-field를 추가할 수 있다.  
예를 들어, service (text) 타입이 있다면 service 필드에 multi-field를 추가하여 service.keyword (keyword)와 같은 필드를 추가할 수 있다. 이렇게 설정하면 service 필드에서는 부분 검색을, service.keyword 필드에서는 keyword 검색을 할 수 있다. 의도하지 않은 부분 검색을 피할 수 있다.  
keyword 필드의 경우 Index pattern에서 `aggregatable` 속성이 활성화된다. 이를 통해 Kibana aggregation을 통해 grouping 등의 기능을 활용해볼 수 있다.  

## 마지막 단계: Index template 수정 후 index rollover

Index template을 수정해도 이미 생성된 index들의 mapping은 변경되지 않는다. 다음 index가 rollver 되어 생성되기를 기다리거나, rollover API를 호출해야 한다.

index rollover에 대한 내용은 다음 포스팅에서 소개하겠다.

