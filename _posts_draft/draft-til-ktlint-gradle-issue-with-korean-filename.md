---
layout: post
title: "[TIL] ktlint-gradle 플러그인의 한글 파일이름 문제"
categories: [Kotlin]
tags: [Kotlin, Gradle, Ktlint]
---

ktlint-gradle 9.4.1: O

ktlint-gradle 10.0.0: ?

ktlint-gradle 10.1.0~10.3.0: X

Gradle 7 사용하려면 ktlint-gradle 10.0.0 이상을 사용해야 한다

[Deprecated Gradle features were used in this build, making it incompatible with Gradle 7.0](https://github.com/JLLeitschuh/ktlint-gradle/issues/395)

이슈 재현도 잘 안됨. 간헐적인 듯.

```
* What went wrong:
Execution failed for task ':runKtlintCheckOverTestSourceSet'.
> A failure occurred while executing org.jlleitschuh.gradle.ktlint.worker.KtLintWorkAction
   > /path/한글파일이름.kt (No such file or directory)
```

아래는 ktlint-gradle 10.2.1 버전부터 fix된 이슈. 한글 파일이름 문제는 반대로 10.3.0에서 발생하기 시작한 것을 보면, 관련이 있을 듯 하다 (`No such file or directory` error)

<https://github.com/JLLeitschuh/ktlint-gradle/issues/539>

ktlint 관련 글들: <https://hojongs.github.io/tags/ktlint/>
