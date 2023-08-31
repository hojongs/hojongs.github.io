---
layout: post
title: "Gradle Test Fixtures Plugin 소개"
categories: [Test code]
tags: [Testing, Fixture]
---

## 문제 상황

- Gradle 멀티 모듈 프로젝트를 생각해보자
- 하나의 모듈에서 테스트 코드에서 사용할 용도로 test sourceSet에 테스트 객체 builder 클래스를 만든다
- 위 모듈을 의존하는 모듈들에서는 해당 builder 클래스를 사용할 수 없다. (test sourceSet의 클래스들은 jar 파일(컴파일된 모듈)에 포함되지 않기 때문)

## 해결방법 및 또다른 문제점

- builder 클래스를 각 모듈 test sourceSet에 복사 -> 중복 코드 관리 어려움
- 테스트용 모듈을 만들고 각 모듈에서 이 모듈을 의존하게 함 -> 꽤 괜찮은 방법. 그나마 단점은 대상 클래스와 builder 클래스의 경로가 멀어짐.

## java-test-fixture

- testFixtures sourceSet이 생김
- test sourceSet은 자동으로 testFixtures sourceSet에 의존함
  - `test` -depends-> `testFixtures` -depends-> `main`
- test-fixtures jar 파일이 생김 -> 컴파일된 testFixtures sourceSet의 클래스들
- 다른 모듈의 testFixtures 클래스들을 의존하려면 아래와 같이 선언
  ```kotlin
  dependencies {
    testImplementation(testFixtures(project(":domain")))
  }
  ```

## testRuntimeOnly

- 모듈에 testRuntimeOnly로 추가된 의존성은 testRuntimeClasspath에 추가됨
- 다른 모듈이 위 모듈을 의존할 때, runtimeClasspath 의존성은 전이됨(transition)
  - 하지만 testRuntimeClasspath 의존성은 전이되지 않음
- h2 DB 의존성과 같이 각 모듈의 testRuntimeClasspath에 전이되기를 원하는 의존성들이 있을 수 있음
- 이런 경우 testFixturesRuntimeClasspath에 의존성을 추가하면 됨
  - 다른 모듈에서 `testRuntimeOnly(testFixtures(...))`와 같이 선언하면 됨

## Conclusion

- `java-test-fixtures` Gradle 플러그인을 통해 각 모듈의 test fixture들을 관리할 수 있다
- Multi-module 프로젝트 test 환경 구성에 유용하다
- main 코드와 test fixture를 동일한 모듈에서 관리할 수 있다

## References

- https://toss.tech/article/how-to-manage-test-dependency-in-gradle
- https://docs.gradle.org/current/userguide/java_testing.html#sec:java_test_fixtures
