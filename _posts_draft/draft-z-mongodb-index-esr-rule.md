---
layout: post
title: "[TIL] MongoDB: ESR rule for compound index"
categories: [MongoDB]
tags: [MongoDB, Database, NoSQL]
---

<https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/>

find query

```json
{
    // ...
    "filter": {
        "status": "NOT_RESERVED_YET",
        "_id": { "$gt": "ObjectId('0000000012345b0df8000000')" },
        "created_at": { "$lt": "new Date(1684447876855)" } 
    },
    "sort": { "_id": 1 },
    // ...
}
```

index (as-is)

```json
{
    "key" : {
            "status" : 1,
            "created_at" : 1,
            "id" : 1
    }
}
```

index (to-be)

```json
{
    "key" : {
            "status" : 1,
            "_id" : 1,
            "created_at" : -1
    }
}
```
