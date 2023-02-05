---
layout: post
title: "Ktlint Gradle 플러그인 비교 (ktlint-gradle vs kotlinter-gradle)"
categories: [Server development]
tags: [Kotlin, Gradle]
---

Gradle 프로젝트에서 ktlint를 사용할 때는 보통 gradle plugin을 사용한다. ktlint 플러그인 중 가장 많이 사용하는 플러그인 2개 ktlint-gradle, kotlinter-gradle를 소개하고 비교한다.

ktlint는 Javascript의 eslint처럼 Kotlin의 lint* tool이다.

> lint?: 작성한 코드의 줄 길이, 들여쓰기(indent), new line of end of the file 등의 코딩 스타일들을 rule로 관리하는 툴

Gradle project에서는 plugin을 사용하면 ktlint를 별도로 설치할 필요가 없다.

> Gradle plugin?: <https://docs.gradle.org/current/userguide/plugins.html>

대표적으로 ktlint Gradle plugin은 아래 두 개가 있다. 최근에 ktlint-gradle에서 kotlinter-gradle로 옮겼는데, 그 이유를 설명한다.

- ktlint-gradle: <https://github.com/JLLeitschuh/ktlint-gradle>
- kotlinter-gradle: <https://github.com/jeremymailen/kotlinter-gradle>

여담으로, 다양한 lint tool들이 있고 그 중에는 여러 언어를 지원하는 툴들도 있는데 Kotlin에서는 보통 ktlint를 사용하는 것 같다.

# ktlint-gradle

아래 내용은 모두 README에서 찾을 수 있다

<https://github.com/JLLeitschuh/ktlint-gradle/blob/main/README.md>
<https://pinterest.github.io/ktlint/>

## How to set up

```kotlin
plugins {
  id("org.jlleitschuh.gradle.ktlint-idea") version "<current_version>"
}

ktlint {
    // configure
}
```

## How to run

```shell
./gradlew ktlintCheck
./gradlew ktlintFormat
```

## Intellij project 설정과 동기화

ktlint의 설정과 Intellij의 설정이 다른 경우가 있다. 예를 들면 multiline string의 indent size가 다를 수 있다.  
그럴 때는 아래 task를 실행하면 

> multiline string?: <https://kotlinlang.org/docs/java-to-kotlin-idioms-strings.html#use-multiline-strings>


```shell
./gradlew ktlintApplyToIdea
```

## Pre-commit hook 등록

```shell
./gradlew addKtlintCheckGitPreCommitHook
./gradlew addKtlintFormatGitPreCommitHook
```

ref: https://github.com/JLLeitschuh/ktlint-gradle/tree/v11.1.0#additional-helper-tasks

## Tips

### ktlint version config

현재 ktlint-gradle 최신 버전은 11.1.0이고, ktlint 최신 버전은 0.48.2이다.  
하지만 ktlint-gradle:11.1.0의 default ktlint 버전은 0.43.2로 꽤 낮다. (그리고 0.48.2는 아직 사용 못하는 것 같다.)

> ref: <https://github.com/JLLeitschuh/ktlint-gradle/releases/tag/v11.0.0>

그래서 아래와 같이 ktlint 버전을 지정하여 최신 버전을 사용할 수 있다.

```kotlin
ktlint {
    version.set("0.48.1")
}
```

### Version compatibility with Kotlin, Gradle, Ktlint

ktlint-gradle 혹은 ktlint 버전을 올릴 때 Kotlin, Gradle의 버전을 올려야 할 때도 있다.
별도로 matrix로 정리된 것은 없는 것 같고, CHANGELOG에서 검색하는 게 제일 빠른 것 같다.

<https://github.com/JLLeitschuh/ktlint-gradle/blob/main/CHANGELOG.md>

# kotlinter-gradle

<https://github.com/jeremymailen/kotlinter-gradle/blob/master/README.md>

## How to set up

```kotlin
plugins {
  id("org.jmailen.kotlinter") version "<current_version>"
}
```

## How to run

```shell
./gradlew lintKotlin
./gradlew formatKotlin
```

## Intellij project 설정과 동기화

별도로 없는데, 아래 editorconfig 파트를 참고하자.

## Pre-commit hook 등록

```shell
./gradlew installKotlinterPrePushHook
```

## Tips

### ktlint version config

ktlint-gradle 대비 거의 최신 버전의 ktlint를 사용하고 있으므로, 별도로 설정할 필요는 없다. (kotlinter-gradle 3.12.0 버전 기준으로, ktlint 0.47.1 버전을 사용하고 있다.)  
버전 설정 방법은 설정이 좀 긴데, README를 참고하자.

### version compatibility with Kotlin, Gradle

아래에 matrix로 정리되어 있다.

<https://github.com/jeremymailen/kotlinter-gradle#compatibility>

# Editorconfig

<https://editorconfig.org/>

Editorconfig는 특정 editor에 의존성이 없는 general한 설정 파일이다.  
ktlint 전용 설정 파일을 통해 프로젝트의 lint rule을 관리할 수도 있겠지만, 이럴 경우 intelliJ, VSCode, Vim 등 사용하는 에디터에 따라 적용 결과가 다를 수 있다.

Editorconfig 파일로 설정을 관리할 경우 이러한 문제를 해결할 수 있다. (Editorconfig를 추천한다.)  
editorconfig 파일 사용법은 외부 레퍼런스로 남기겠다.

- ktlint: <https://pinterest.github.io/ktlint/install/cli/#rule-configuration-editorconfig>
  - ktlint에서는 editorconfig 파일 생성 명령어를 지원한다.
    ```shell
    ktlint generateEditorConfig
    ```
- kotlinter-gradle: <https://github.com/jeremymailen/kotlinter-gradle/tree/3.13.0#editorconfig>
- <https://jojoldu.tistory.com/673>

## ktlint with editorconfig troubleshooting

### ktlint 0.48.1

`disabled_rules` 설정이 deprecated 되었다. `ktlint_disabled_rules` 설정을 사용해야 한다.

### ktlint-gradle 11.1.0

- 이와중에 ktlint-gradle 11.1.0에서 ktlint 0.48.1 버전 사용 시 warning log가 출력되는 이슈가 있다고 한다.
- <https://github.com/JLLeitschuh/ktlint-gradle/issues/622>

### ktlint 0.46.0 ~ 0.47.0

- editorconfig disabled_rules 설정이 무시되는 버그가 있다 (...)
<https://github.com/pinterest/ktlint/pull/1671>
- 0.48.0 버전에서 해결되었고, workaround는 아래와 같다.

```kotlin
// build.gradle.kts
ktlint {
    version.set("0.46.0")
    disabledRules.add("import-ordering")
}
```

### ktlint 0.45.0

```
Could not initialize class com.pinterest.ktlint.core.internal.EditorConfigLoaderKt
```

이런 에러가 발생했던 것 같은데, 0.45.2 버전하면 해결됐다.

# Conclusion

ktlint-gradle과 kotlinter-gradle 둘다 사용할 수 있는 플러그인이다. 하지만 ktlint-gradle은 최근(2022.01)에 maintainaner의 리소스 부족 이슈가 있는 듯하다.

<https://github.com/JLLeitschuh/ktlint-gradle/issues/569>

이것이 의미하는 바는, ktlint-gradle과 함께 사용하는 ktlint의 버전이 높을수록 호환성 문제가 많이 발생할 수 있다느 의미다.

위 이슈를 멘션한 다른 저장소의 이슈들을 보면, 대체 plugin을 찾고 있다. kotlinter-gradle 등.

- <https://github.com/AdamMc331/AndroidAppTemplate/issues/17>
- <https://github.com/tuskyapp/Tusky/issues/2914>

이 포스트가 ktlint-gradle 플러그인을 사용하고 있는 사람들에게 참고가 되길 바란다.  
우리 팀에서는 ktlint-gradle을 계속해서 사용해오다가, 최근에 Gradle 등 버전을 올리면서 여러 가지 이슈를 겪고 결국 kotlinter-gradle로 플러그인을 변경하였다.

spotless라는 대안도 있는 것 같은데, 고려해보진 않았다. 저장소를 봤을 때는 maintenance가 더 잘 되지 않을까하는 기대가 있다.

<https://github.com/diffplug/spotless>
