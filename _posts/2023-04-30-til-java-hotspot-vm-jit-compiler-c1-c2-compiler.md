---
layout: post
title: "[TIL] JIT compiler in Java HotSpot VM, and C1 vs C2 compiler mode"
categories: [JVM]
tags: [Java, JVM, JIT, TIL, HotSpot VM, Performance Optimization]
---

## JVM C1, C2 compiler

회사 팀원이 갑자기 C1, C2 compiler에 대해 아냐고 질문했다. 사실 JVM의 C1, C2라는 용어는 처음 들어보았다. 그래서 조금 찾아보았고 매우 간단하게 이해한 내용을 정리해본다. (용어 정의 정도)

> 정말 간단하게 알아봤으므로 갑자기 끝나도 놀라지 말자..

## Understanding Java JIT Compilation

Oracle에 2014년에 쓰여진 아래 문서를 찾았다.

<https://www.oracle.com/technical-resources/articles/java/architect-evans-pt1.html>

JVM의 종류가 여러 가지 있는데, 위 문서는 Oracle의 HotSpot VM에 대한 내용이다. (본인은 HotSpot만 사용해보았다) 정확히는 JITWatch를 소개하는 내용인 것 같은데 JIT compiler, C1, C2에 대한 내용도 소개되어 있다. 내용을 읽어보자. 참고로 이 포스트에서는 내용 전체를 번역하기보다는 관심있는 부분만 point out 해볼 예정이다.

---

HotSpot VM은 JIT compiler를 내장하고 있다.

> JIT compiler는 Just-In-Time compiler의 약자로 runtime에서 bytecode를 native machine code로 변환하는 compiler이다.
> Java source code는 bytecode(class files)로 컴파일되고, JIT compiler는 이 bytecode를 native machine code로 변환한다.

### Basic JIT Compilation

HotSpot VM은 실행되는 메소드들을 모니터링하다가, 어떤 메소드가 특정 조건(e.g. 호출 빈도)을 만족하면 이 메소드를 컴파일한다. 이를 hot method라고 부르고, 이 컴파일은 별도의 JVM thread에서 이루어진다.

즉, JVM 실행 자체와 별도로 JIT compiler가 실행되는 것이다. JIT compiler가 컴파일하고 있을 때 JVM은 interpreted 버전을 실행한다. compile이 완료되면 JVM은 컴파일된 method, 즉 interpreted bytecode가 아니라 native machine code를 실행하게 될 것이다.

즉, JVM의 기본적인 구조는 bytecode를 interpret하여 실행하는 것이지만(Java interpreter) JIT compiler가 bytecode를 native machine code로 컴파일하여 실행 속도를 가속화하는 테크닉이 구현된 것이다.

`-XX:+PrintCompilation` flag를 사용하면 JIT compile 로그를 볼 수 있나보다.

---

### Some JIT Compilation Techniques

JIT compiler는 컴파일을 하면서 코드를 최적화하는데, 가장 일반적인 최적화 테크닉이 inlining이라고 한다. JIT compiler는 35 바이트 이하의 메소드들을 inlining하려고 시도한다고 한다. inline을 하면 메소드 호출 시 새로운 stack frame이 만들어지지 않는 이점이 있다.

이 외에도 다양한 테크닉들이 있고 이를 통해 JIT compiler는 C/C++가 컴파일된 코드에 준하는 성능을 낼 수 있다고 한다.

### Compilation Modes (드디어 C1 C2!)

> 이전까지는 C1, C2를 설명하기 전의 배경지식으로서 JIT compiler 관련한 내용들이었다. 이제 C1, C2 컴파일러에 대해 알아보자.

HotSpot VM에는 2개의 JIT compile 모드가 있다고 한다: C1, C2. C1은 빠른 startup이 필요한 application에 적합한데, GUI application이 적절한 예시이다. 반대로 C2는 장기간 실행되는, 서버 사이드 애플리케이션에 적합하다. (사실 이 설명만으로는 잘 와닿지 않는다. 아래를 보자.)

두 컴파일러 모드는 서로 다른 테크닉을 사용한다. 그래서 같은 메소드에 대해서 machine code 출력물이 다르다. 일부 최신 Java 7 이전에는 둘 중 하나의 모드만 사용할 수 있었지만 이후부터는 함께 사용할 수 있다. 이를 위해 `tiered compilation`이라고 불리는 새로운 기능이 함께 추가되었다.

`tiered compilation`은 JVM 시작 시 C1 compiler 모드를 사용한다. 이를 통해 애플리케이션 시작 시간을 줄일 수 있다. 애플리케이션이 적절히 warm-up된 후 C2 compiler 모드로 전환한다. C2 compiler를 통해 더 최적화된 machine code를 생성하고 더 높은 성능을 낼 수 있다. Java SE 8부터는 `tiered compilation`이 기본 설정이다.

> **즉, C1 compiler는 최적화보다는 컴파일 속도를 최대화하고 C2 compiler는 컴파일 속도보다는 최적화를 최대화하는 것을 목표로 한다고 이해된다.**

> 적절히 warm-up이라 함의 좀더 구체적인 의미는 C2 compile을 위한 충분한 profiling 정보가 확보됨을 의미하는 것으로 보인다. (하단 reference 2 Baeldung 문서의 3.1 section 참고)

원문에서 이후에는 JIT compile 로그를 통해 String::hashCode() 메소드가 어떤 컴파일러(C1)로 컴파일 되었는지 확인해보는 내용이 서술되어 있다.

## Conclusion

위 문서에서 C1, C2의 간단한 정의에 대해 알아보았다. 막상 읽어보니 내용이 많지는 않았다. 본문 내 링크도 깨져 있어서 내용을 더 읽어보려면 새로운 문서를 찾아야 할 것 같다.

알고보니 이 의문의 시작은 아래 영상이었다. 아직 보지는 못했지만, 아래 영상에서 C1, C2 컴파일러 관련한 좀더 자세한 내용을 다룬다. 관심있는 사람들은 시청해보도록 하자.

<https://if.kakao.com/2022/session/35>

> 정말 간단하게 쓰려고 했으나 글이 또 계획보다 길어진 것은 숨길 수 없는 비밀이다...
여담으로 앞으로는 TIL 포스팅도 가끔씩 해볼 예정이다.

## Read more

위 내용에는 포함되지 않았지만, 추가로 발견한 관련 포스트들도 공유한다.

- <https://forums.oracle.com/ords/apexds/post/jvm-c1-c2-compiler-thread-high-cpu-consumption-8752>
- 2: <https://www.baeldung.com/jvm-tiered-compilation>
