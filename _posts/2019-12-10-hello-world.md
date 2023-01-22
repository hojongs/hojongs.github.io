---
layout: post
title: 기술 블로그 시작
categories: [Random]
tags: []
redirect_from:
  - /hello-world/
  - /2019/12/10/hello-world.html
---

글 작성 테스트를 위한 포스트입니다.

Github Web에서 작성은 불편해서, Local IDE에서 작성해야겠습니다.

줄바꿈을 하려면, 줄 사이에 여백 라인이 필요합니다.

<br>

타 블로그 사이트들에 대한 github.io의 차별점은 아래와 같습니다.
* 포스팅이 commit log로 남는다. ~~뿌듯함을 느낄 수 있다~~
  * 단, fork한 repository일 경우 profile의 contributions에는 기록되지 않는다
    * [Why are my contributions not showing up on my profile?](https://help.github.com/en/github/setting-up-and-managing-your-github-profile/why-are-my-contributions-not-showing-up-on-my-profile)
    * fork 외에 `git config의 email이 github 계정이 연동되어 있어야함`, `master 브랜치에 push`의 조건이 있다
  * 하지만 PR을 여는 것은 contributions에 기록된다... ~~그렇게까지 할 바에야~~
* 글을 수정해도 글의 변경 이력이 남는다.
* 작성 날짜를 임의로 지정할 수 있다. (수동으로 관리해야 한다는 건 단점일 수 있음)
* 제3자가 PR을 통해 글에 기여할 수도 있다.

---

* 코드 블록
  ```kotlin
  package com.hojongs
  
  import com.hojongs.MyClass
  
  fun f() {
    println("Hello, world!")
  }
  ```

```python
print('hi')
```

<br>

틀린 내용 지적 / 질문 환영!
