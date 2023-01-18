---
layout: post
title: UUID란 무엇인가
toc: true
---

* UUID는 Universally unique identifier의 약자로서, 정보 식별을 위하여 사용되는 식별자이다
* 128-bit 숫자로 이루어져 있으며, xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx 형식으로 표현한다
* UUID의 장점 중, 데이터들이 나중에 단일 DB로 통합되거나, 같은 채널에서 전송되더라도 식별자가 중복될 확률이 매우 낮다는 점이 있었다

---

# Motivation of the posting

## Personal Motivaton

- 지금 다니고있는 회사의 DB Schema를 보았더니, Table의 ID Column의 데이터 타입이 UUID였다
- Integer와 비교했을 때 UUID를 선택한 이유가 무엇인지 궁금했다
- 심지어 UUID 자체를 처음 접했기 때문에, UUID의 정의 및 개념을 공부하는 것이 먼저였다
- 그리하여, 다음에 대해 알아보고자 공부하게 되었다
  - UUID가 무엇인지
  - DB에서 Primary Key로 UUID를 사용하면 어떤 이점이 있는지 

## Database Primary Key and ID column

- Database에 데이터를 저장할 때, 데이터 식별을 위해 Primary Key를 사용하고 있다
- Primary Key는 성능적 이점을 위해 Int Data type과 ID라는 이름을 사용하였다
- 데이터들은 같은 ID를 가져서는 안된다. 이러한 constraint를 지키기 위한 가장 간단한 방법은 다음과 같다.
  - 데이터 생성 시, DB가 자동으로 1부터 순서대로 ID를 할당해주도록 하는 것이다.
  - 이 방법은 게시판의 데이터 관리 시 유용하다

## Using UUID as a Primary Key

- 그리고 또다른 방법 중 하나로, UUID를 사용하는 방법이 있다
  - 이를 위해 Exposed 라이브러리에서는 UUIDTable 클래스를 제공해주기도 한다
- incremental int id 대신 UUID를 사용하면, 여러 가지 이점을 얻을 수 있다
- 본 글은, 아래와 같은 내용들을 조사하기 위해 작성되었다
  - UUID의 기본적인 정의 및 개념
  - Database에서 데이터 식별을 위해 UUID를 사용했을 때 
- [UUID - Wikipedia](https://en.wikipedia.org/wiki/Universally_unique_identifier)를 읽어보았으며, 위 내용에 대한 간단한 수준의 답을 얻을 수 있었다

> 모두 읽기에는 시간이 부족하여, 일부 관심있는 내용들만 읽고 정리하였다

# UUID

- UUID는 128-bit 숫자로, 정보 식별에 사용됨
- Microsoft의 S/W에서는 GUID라고 불림
- standard mehod로 생성 시, UUID는 실용적인 용도에서 (충분히) 고유함
- UUID의 고유성은 다른 numbering scheme과 달리, 중앙 등록 기관이나, 생성된 UUID 사이의 조정(coordination)에 의존하지 않음
- UUID가 중복될 확률이 0은 아니지만 → 무시해도 될 정도로 0에 가까움

<br>

- 그러므로, 식별자가 거의 중복되지 않는다는 확신과 함께, 누구나 UUID를 생성하고 사용할 수 있다
- 독립적인 일행들(parties)의, UUID를 가진 정보는 무시할만한 중복 가능성과 함께 나중에 단일 DB로 통합되거나 같은 채널에서 전달(transmit) 될 수 있음
- UUID는 널리 채택되어있고, 많은 컴퓨팅 플랫폼은 UUID 생성 및 UUID 텍스트 표현의 파싱을 지원함

# History

...

# Standards

...

# Format

- canonical 텍스트 표현에서, UUID는 16 octets가 32 hex 숫자로 표현되고, hypen(-)으로 구분된 5개의 그룹으로 이루어짐
- 8-4-4-4-12 형식
- `123e4567-e89b-12d3-a456-426655440000`
- `xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx`
- M은 4 bit로, version을 의미
    - 위 예제에서 M=1, version=1
- N은 1~3 bit로, variant를 의미
    - 위 예제에서 N=a(=10xx), variant=1
- 5개의 그룹은 순서대로 아래를 의미함

## UUID record layout

|Name|Length (bytes)|Length (hex digits)|Contents|
|----|--------------|-------------------|--------|
|time_low|4|8|integer giving the low 32 bits of the time|
|time_mid|2|4|integer giving the middle 16 bits of the time|
|time_hi_and_version|2|4|4-bit "version" in the most significant bits, followed by the high 12 bits of the time|
|clock_seq_hi_and_res clock_seq_low|2|4|1 to 3-bit "variant" in the most significant bits, followed by the 13 to 15-bit clock sequence|
|node|6|12|the 48-bit node id|

> 이후 내용은 읽지 못했으므로 여기서 마침

## Conclusion

- UUID는 128-bit로 이루어진, "실용적인 측면에서 충분히 고유한" universal 식별자이다
- UUID를 사용했을 때의 이점들 중 몇 가지를 뽑자면 다음과 같다
  - UUID의 고유성은 중앙 등록 기관(예를 들면 데이터베이스 서버) 등에 의존되지 않고, standard method를 통해 독립적으로 생성 가능
  - 별도로 분리되어 있던 데이터들을 통합하거나, 하나의 채널에서 전송하더라도 충돌이 발생하지 않는다
  - UUID는 널리 채택되어 있고 많은 컴퓨팅 플랫폼들에서 UUID 생성, 파싱을 지원하고 있음
    - 예를 들면 Exposed의 UUIDTable class이 될 것 같다
