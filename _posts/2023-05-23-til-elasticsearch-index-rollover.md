---
layout: post
title: "[TIL] Elasticsearch: How-to index rollover"
categories: [Elasticsearch]
tags: [Elasticsearch, TIL]
---

지난 포스팅 </posts/til-kibana-index-template/> 에서 index template을 수정하였다. (Kibana에서 특정 column의 값이 full-text search 되지 않는 문제 해결)

이렇게 template을 수정해도 이미 생성된 index에는 영향을 미치지 않는다. 다음 index가 생성(rollover)될 때부터 변경된 결과를 확인할 수 있다.

지금 들어오는 로그(document)들부터 즉시 변경사항이 적용되도록 하기 위해, index를 수동으로 rollover시키는 방법이 있다.

## Elasticsearch index rollover

<https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-rollover-index.html>

`POST /<rollover-target>/_rollover/`

### API Authentication

이 API를 사용하려면 물론 API 인증이 필요한데, ES는 basic authentication을 사용한다. (Authorization HTTP header)

<https://www.elastic.co/guide/en/elasticsearch/reference/current/http-clients.html>

```
Authorization: Basic <TOKEN>
```

`<TOKEN>`에는 `base64(USERNAME:PASSWORD)` 값이 들어간다.

curl example:

```sh
curl -X POST -u hojongs:mypassword '.../_rollover/'
```

## Conclusion

이렇게 index rollover 후, 변경된 template이 적용된 index가 바로 생성되고 변경사항이 적용되는 것을 확인할 수 있다.
