---
layout: post
title: "[TIL] Gradle: dependency cache path in Home directory"
categories: [Gradle]
tags: [TIL, Gradle, Jar, Multi-module]
---

현재 회사에서 소속 팀은 서버 플랫폼 팀이다. 우리 팀에서는 내부 라이브러리를 개발하고 maintain하는 일도 하고 있다. 이 라이브러리들은 Nexus에 배포하고 각 서버 저장소에서 "의존성으로서" 의존하여 사용된다.

Gradle은 빌드 속도를 높이기 위해 이러한 의존성들을 로컬에 cache한다. cache key는 artifact id, version인데 이에 따라 주의할 점이 하나 있다.

한 번 publish한 artifact를 변경하여 다시 publish해도, 빌드 환경에 이미 캐시가 되어있으면 반영되지 않을 수 있다는 것이다.

그래서 위 문제를 해결하기 위해 이 cache의 path가 어디인지 알아보게 되었다.

# Gradle dependency cache

자세한 내용은 아래에서 찾을 수 있다.

<https://docs.gradle.org/current/userguide/directory_layout.html>

dependency cache들은 Gradle wrapper 여러 버전들에 공유된다. 따라서 modules-2 디렉토리에 저장된다.

실제로 경로에 가보니 아래와 같은 구조로 되어있었다.

.gradle/caches/modules-2/files-2.1/{package}/{artifact}/{version}

아래는 참고.

> Files in shared caches used by the current Gradle version in caches/ (e.g. jars-3 or modules-2) are checked for when they were last accessed. Depending on whether the file can be recreated locally or would have to be downloaded from a remote repository again, it will be deleted after 7 or 30 days of not being accessed, respectively.

# 의존성 캐시 문제 해결

배포한 라이브러리 버전에 이슈가 있어서 수정하여 다시 배포했다 (같은 버전으로)

그런데 슬프게도 이미 build agent에 bug 버전이 캐시되어버렸다

버전을 바꾸는 방법도 있었지만 더 번거롭기도 하고, 위 방법으로 해결이 가능한지 확인해보고 싶었다

각 build agent에서 해당 라이브러리의 의존성 캐시를 제거하고, 다시 빌드했더니 의존성이 refresh되어 정상적으로 빌드되었다

---

TIL: 위 bug는 컴파일 에러를 발생시켰기 때문에 다행이었지만 런타임 에러의 경우 바로 인지하기 어렵기 때문에, 동일 버전을 다시 publish하는 일은 최대한 지양해야겠다

## TMI

### Nexus repository

보통 회사 내부에서 사용하는 라이브러리(Maven artifact)들은 Nexus에 publish하고 저장&관리한다 (Nexus가 아니면 다양한 cloud provider에서 제공하는 package manager를 사용할 수 있다)

- <https://www.sonatype.com/products/sonatype-nexus-oss>
- <https://github.com/sonatype/nexus-public>

### Nexus Helm chart

Nexus를 K8s cluster에 배포하고 싶다면 아래 Helm chart를 사용하면 될 것 같은데, 2022/10/24부터 maintain 하지 않는듯 하다. (2023~이라고 써있는데 2022인 듯..)

<https://github.com/sonatype/nxrm3-helm-repository/tree/main/nexus-repository-manager>
