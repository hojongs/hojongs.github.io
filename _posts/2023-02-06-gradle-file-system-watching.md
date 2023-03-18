---
layout: post
title: "Gradle file system watching: 동작방식 및 사용법"
categories: [Server development]
tags: [Gradle, Productivity]
redirect_from:
  - /posts/gradle-file-system-watching/
---

File system watching:

- incremental build*를 매우 가속화하는 기능
- 기본적으로 Gradle은 build 시마다 file system을 polling함 (매번 확인함)
- File system watching을 활성화하면, Gradle이 build 사이에 file system에 대해 학습한 것들을 메모리에 저장하고 polling하지 않음
- 이는 Disk I/O 횟수로 굉장히 감소시킴 - 이 Disk I/O는 이전 빌드로부터 바뀐 것이 무엇인지 확인하기 위한 비용임

incremental build: <https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:up_to_date_checks>

## File system watching and Gradle versions

- Gradle 6.5 → experimental
- Gradle 6.7 → stable (but disabled by default yet)
- Gradle 7.0 → enabled by default

## How to work file system watching

- Gradle이 task 실행 여부를 결정하려면 지난 build로부터 input/output 파일이 변경되었는지 확인해야 함
- Gradle daemon은 현재 빌드가 끝날 때까지의 파일 시스템 정보를 메모리에 저장함 -> 이게 바로 Gradle의 virtual file system

- file system watching이 없으면 daemon은 build가 끝날 때마다 수집한 그 정보들을 제거함
- 하지만 활성화되어 있으면 daemon은 OS로부터 disk의 변경사항을 알림받음
  -> file system의 변경사항을 watch함
  -> virtual file system에 저장된 정보 중 변경되지 않은 파일을에 대한 부분은 재활용할 수 있음 -> Disk I/O를 수행하지 않음

## How to enable file system watching

```shell
./gradlew build --watch-fs

## to disable
./gradlew build --no-watch-fs
```

or

```properties
## Gradle>=6.7
org.gradle.vfs.watch=true
```

### Enable verbose logging

```properties
org.gradle.vfs.verbose=true
```

```shell
./gradlew build -Dorg.gradle.vfs.verbose=true
```

## Reference

- <https://docs.gradle.org/current/userguide/file_system_watching.html>
- <https://blog.gradle.org/introducing-file-system-watching>
