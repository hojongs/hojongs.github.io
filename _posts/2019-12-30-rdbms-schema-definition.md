---
layout: post
title: RDBMS Schema란 무엇인가
categories: [Database]
tags: [Kotlin, Flyway, Exposed(Kotlin)]
redirect_from:
  - /posts/rdbms-schema-definition/
  - /2019/12/30/rdbms-schema-definition.html
---

- 회사에서 참여하고 있는 프로젝트가 Exposed SQL Framework와 Flyway DB Migration Tool를 사용하고 있다
- 이 두 Tool들이 DB Schema를 생성하는 것을 보고, Schema가 구체적으로 무엇인지 호기심이 생겼다
- PostgreSQL 문서에 Schema에 대한 내용을 읽고 정리해보았다
    - [PostgreSQL: Documentation: 9.5: Schemas](https://www.postgresql.org/docs/9.5/ddl-schemas.html)

# Schemas

- Database는 하나 이상의 Schema를 가질 수 있다
- Schema는 Table, Data type, Function, Operator과 같은 Object들을 가질 수 있다
- 서로 다른 Schema가 같은 오브젝트 이름을 가지더라도, 충돌은 발생하지 않는다
    - schema1과 schema2가 동시에 mytable object를 가질 수 있다
- Database와 달리, Schema는 rigidly 분리되어 있지 않다
    - client connection가 오직 하나의 database에만 접근할 수 있는 반면,
    - database 내의 모든 schema에 접근할 수 있다 (물론 privilege를 가지고 있다면)

## Schema 사용 이유

- 여러 유저들이 하나의 데이터베이스를 충돌(interfering)없이 사용하기 위해
- 데이터베이스 오브젝트들을 그룹화하여 논리적으로 나눔 → 더 효율적인 관리를 위해
- 제3 응용들이 각각의 schema를 사용 → 오브젝트 이름 충돌 방지를 위해

> 중첩될 수 없다는 점만 제외하면, Schema는 OS-level의 디렉토리와 비슷한 개념이다

## PostgreSQL Schema

- Create, Drop Schema

```sql
    CREATE SCHEMA myschema;
    DROP SCHEMA myschema;
    DROP SCHEMA myschema CASCADE;
```

- Schema access

```sql
    myschema.mytable
    mydb.myschema.mytable // more general syntax
```

- Schema.table 생성

```sql
    CREATE TABLE myschema.mytable ( ... );
```

### The Public Schema

- 기본 schema는 public임
- schema 명시 없이 table 생성 시 public schema에 생성됨

```sql
    CREATE TABLE mytable -- == CREATE TABLE public.mytable
```

# Conclusion

- Schema는 Database의 구성요소로서, 충돌을 방지하기 위해 사용되는 논리적 그룹이었다
- PostgreSQL에서 public schema는 기본적으로 생성되는 schema였다
- Database와 대비되는 Schema의 이점은, schema들이 공통 schema를 공유하여 사용할 수 있을 것 같다는 생각이 들었다
- Database가 물리적으로 분리된 오브젝트 그룹이라면, Schema는 논리적으로 분리된 그룹으로 설명 할 수 있을 것 같다 (HDD와 Directory처럼)