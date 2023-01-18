---
layout: post
title: Reactor Reference Reading
toc: true
redirect_from:
  - /reactor-reference/
---

* Spring Webflux에 내장되어 있는 Reactor를 사용하기 위해 Reactor Reference를 읽으면서 공부하게 되었다 
* Reactor는 Rx와 같은 Reactive 패러다임의 구현체이며, Reactive Streams sepc의 구현도 포함하고 있는 Java 라이브러리이다
* Reactor reference를 읽으면서 공부해보았고, 나름 번역하여 본 글을 작성해보았다

# 1. About the Documentation

* `생략`

# 2. Getting Started
## 2.1 Introducing Reactor

* Reactor = fully non-blocking reactive programming foundation for JVM
* 효과적인 demand management 또한 제공 ("backpressure"를 관리하는 형식으로)
* Java 8 함수형 API와 잘 결합한다
* composable 비동기 sequence API - Flux, Mono를 제공한다
* Reactive Streams 명세(spec)을 널리 구현하였다

<br>

* Non-blocking IPC 지원 (reactor-netty project)
* Microservice 아키텍처에 적합한 Reactor Netty는 Backpressure-ready 네트워크 엔진을 제공함
  * HTTP, TCP, UDP를 위한 네트워크 엔진
  * Reactive 인코딩/디코딩 완벽 지원

#### Keyword

|Keyword|Description|
|-------|-----------|
|Non-blocking|I/O 처리 방식, Blocking의 반대|
|Reactive Programming (Paradigm)|뒤에서 언급 예정|
|Backpressure|뒤에서 언급 예정|
|함수형 (Functional)|선언형 프로그래밍 패러다임 중 하나|
|Composable|compose 가능한, 뒤에서 언급 예정|
|Reactive Streams|Java Reactive Library 표준 Spec|
|Backpressure-ready network engine|Reactor Netty를 사용해보지 않아서 모르겠음|

## 2.2 Prerequisites

* Java 8 이상 필요
* It has a transitive dependency on org.reactivestreams:reactive-streams:1.0.3.
  
>  Android Support
>
>  Reactor 3 does not officially support or target Android (consider using RxJava 2 if such support is a strong requirement).
>
>  However, it should work fine with Android SDK 26 (Android O) and above.
>
>  We are open to evaluating changes that benefit Android support in a best-effort fashion. However, we cannot make guarantees. Each decision must be made on a case-by-case basis.

## 2.3 Understanding the BOM

* BOM : Bill of Materials
* 이 curated (BOM) 리스트는 아티팩트들을 그룹화함
  * 이러한 아티팩트들의 versioning scheme이 잠재적으로 분기하더라도, 함께 사용하기 위하여
* BOM은 스스로 versing됨
  * codename과 제한자(qualifier)를 통해 release train scheme 사용
* 예시

```
Aluminium-RELEASE
Californium-BUILD-SNAPSHOT
Aluminium-SR1
Bismuth-RELEASE
Californium-SR32
```

* <codename>-<qualifier> 형식
* The codenames represent what would traditionally be the MAJOR.MINOR number. They (mostly) come from the Periodic Table of Elements, in increasing alphabetical order.

<br>

* The qualifiers are (in chronological order):
  * BUILD-SNAPSHOT: Builds for development and testing.
  * M1..N: Milestones or developer previews.
  * RELEASE: The first GA (General Availability) release in a codename series.
  * SR1..N: The subsequent GA releases in a codename series — equivalent to a PATCH number. (SR stands for “Service Release”).

<br>

* 글을 작성하고있는 현재, Reactor Core version는 `3.3.1.RELEASE`, Release Train은 `Dysprosium-SR2`이다.
* Reactor Netty 또한 version은 `0.9.2.RELEASE`임에도 불구하고 Release Train은 `Dysprosium-SR2`임을 확인할 수 있다.

#### Keyword

|Keyword|Description|
|-------|-----------|
|BOM|직역하면 Material들의 명세서, 연관된 아티팩트들의 편리한 버전 관리를 위한 버전 명세방법|

## 2.4 Getting Reactor

### 2.4.1 Maven Installation

`생략`

### 2.4.2 Gradle Installation

#### Keyword

|Keyword|Description|
|-------|-----------|
|Gradle|Build system 중 하나|

#### TODO

* 현재 Gradle이 Maven보다 널리 사용된다고 알고 있다.
* 비교적 빠른 빌드속도 외에 gradle이 널리 사용되는 이유는 뭘까?

### 2.4.3. Milestones and Snapshots

# 3. Introduction to Reactive Programming

* Reactor = Reactive programming paradigm의 구현체

> Reactive Programming = 데이터 스트림과 변화 전화를 고려한, 비동기 프로그래밍 패러다임 <br>
> 따라서 정적 데이터나 동적 데이터 스트림을 쉽게 표현할 수 있다 <br>
> (정적 데이터 = arrays, 동적 데이터 = event emitters) <br>

* Microsoft의 .NET Rx 라이브러리가 Reactive Programming의 최초 구현체이다
* 이후 RxJava가 나왔고, Reactive Streams를 통해 Java Standard가 등장했다
  * Java 표준 명세는, reactive library의 인터페이스와 상호작용 규칙을 명세했다
* 이 인터페이스는 Java 9 Flow class로 integrate 되었다

<br>

* Reactive 프로그래밍 패러다임은 OOP의 옵저버 디자인 패턴의 확장으로 자주 표현된다
* 이러한 라이브러리(?) 모두에 Iterable-Iterator pair에 대한 이중성(duality)이 있다
* 그러므로 주요한 reactive streams pattern을 Iterator design pattern과 비교해볼 수 있다

<br>

* "값에 접근하는 방법(method)이 Iterable 단독의 책임임에도 불구하고 Iterator를 사용하는 것"은 명령형 프로그래밍 패턴이다
* 사실, 언제 sequence의 next() item에 접근할 것인지는 개발자의 선택이다
* Reactive streams에서, 위 pair에 대응되는 것은 Publisher-Subscriber이다.
* 하지만 새로운 값이 사용 가능해졌음을 Subscriber에게 notify하는 것은 Publisher이다
  * 그리고 이 push 측면(aspect)은 reactive의 핵심 key이다
* 또한, push된 값에 적용된 연산(operation)은 (명령적이기보다는) 선언적으로 표현된다
* 프로그래머는 (정확한 control flow보다는) 연산 로직을 표현한다

<br>

* push되는 값에 추가로, error 핸들링과 completion 측면 또한 잘 정의된 방식(manner)으로 다루어집니다(cover)
* Publisher는 새로운 값을 Subscriber에게 push할 수 있지만 (Subscriber의 onNext 호출을 통해)
* error 또는 completion 시그널 또한 보낼 수 있다 (onError, onComplete)
* 이 둘은 sequence를 끝낸다(terminate)

<br>

* 이러한 접근방식(`onNext x 0..N [onError | onComplete]`)은 아주 유연하다(flexible)
* 이 패턴은 값이 없거나, 하나이거나, n개인 경우를 모두 지원한다 (continuing ticks of a clock과 같은 무한 sequence도 포함)

#### Keyword

|Keyword|Description|
|-------|-----------|
|Observer Design Pattern|TODO|
|Imperative programming|명령형 프로그래밍 패러다임, 선언형 프로그래밍의 반대|
|Control Flow|프로그램의 실행 제어 흐름|


## 3.1. Blocking Can Be Wasteful

* 현대 응용들은 거대한 양의 동시 사용자를 가질 수 있고, 
* 현대 하드웨어의 용량은 계속해서 향상되고 있음에도 불구하고 현대 소프트웨어의 성능은 여전히 핵심 관심사이다

<br>

* 프로그램 성능 향상에 대략 두 가지 방법이 있다
  * 병렬화, 더 많은 스레드와 더 많은 하드웨어 자원을 사용한다
  * 효율성 추구, 현재 자원을 유지하면서.

<br>

* 보통, 자바 개발자들은 블로킹 코드로 프로그램을 작성한다
* 이 방법은 성능 병목 지점이 있을 때까지는 괜찮다
* 그리고 비슷한 블로킹 코드를 실행하는 스레드 수를 늘린다.
* 하지만 이러한 자원 활용(utilization)의 scaling은 금방 경쟁(contention)과 동시성(concurrency) 문제를 발생시킨다

<br>

* 더 나쁜 것은, 블로킹이 자원을 낭비한다는 점이다.
* 프로그램이 약간의 latency(특히 DB 요청, 네트워크 호출 등의 I/O로 인한 latency)를 가지는 즉시,
* thread들은 idle 상태가 되어 데이터를 기다리고 있으므로 자원은 낭비된다

<br>

* 그래서, 병렬화 접근방법은 해결방법이 아니었다.
* 하드웨어의 full power를 사용해야 하지만, (하드웨어를 더 효율적으로 활용해야 하지만)
* 리소스 낭비는 이유가 복잡하고 발생하기 쉽다

## 3.2. Asynchronicity to the Rescue?

* 두번째 접근방법 효율성 추구는 리소스 낭비 문제의 해결방법이 될 수 있다
* 비동기 Non-blocking 코드를 통해 execution을, 같은 자원을 사용하는 다른 active task로 switch한 후
* 비동기 프로세스가 끝나면 현재 프로세스로 돌아오도록 할 수 있다

<br>

* 자바는 비동기 프로그래밍의 2가지 모델을 제공한다
  * Callback : 
  * Future : 

`TODO`

## 3.3. From Imperative to Reactive Programming

`TODO`

# 4. Reactor Core Features

`TODO`

## 4.1. Flux, an Asynchronous Sequence of 0-N Items

`TODO`

## 4.2. Mono, an Asynchronous 0-1 Result

`TODO`

## 4.3. Simple Ways to Create a Flux or Mono and Subscribe to It

`TODO`

## 4.4. Programmatically creating a sequence

* 이 섹션에서는, 관련 이벤트들(onNext, ...)을 프로그래밍 방식으로 정의하여, Flux나 Mono를 생성하는 것을 소개한다
* 이러한 모든 방법들은, 공개된 API를 가지는데, 이것들은 event를 발생시켜 sink를 호출한다
* 몇 가지의 sink들이 있는데, 한번 보자

### 4.4.1. Synchronous generate

* 가장 간단한 프로그래밍 방식 Flux 생성은 generate method를 통한 것이다
* 이것은 generator function을 인자로 받는다

<br>

* 이것은 동기 방식의 1:1 방출(emission)을 위한 방식이다.
* 이것이 의미하는 바는, sink는 동기식 Sink이며 콜백에서 sink.next()는 최대 한 번만 호출해야 한다
* error()와 complete()는 옵션이다

<br>

* 가장 유용한 sink는 아마 당신이 state(상태)를 가지도록 하는 sink이다
* 당신은 이 state를 다음에 무엇을 emit할 지 결정하기 위해 sink 사용 시에 참조할 수 있다
* generator 함수는 `BiFunction<S, SynchronousSink<T>, S>`이 되며, `<S>`는 state 객체의 자료형이다
* 당신은 `Supplier<S>`를 초기 state를 위해 제공할 수 있고, 당신의 generator 함수는 각 순회마다 새로운 state를 리턴한다

<br>

* 예를 들어, 당신은 int형을 state로 사용할 수 있다
* Example 11. Example of state-based generate

```java
Flux<String> flux = Flux.generate(
    () -> 0, 
    (state, sink) -> {
      sink.next("3 x " + state + " = " + 3*state); 
      if (state == 10) sink.complete(); 
      return state + 1; 
    });
```

* We supply the initial state value of 0.
* We use the state to choose what to emit (a row in the multiplication table of 3).
* We also use it to choose when to stop.
* We return a new state that we use in the next invocation (unless the sequence terminated in this one).

<br>

* 앞의 코드는 구구단 3단을 생성한다, 그 결과는 아래에.
> 3 x 0 = 0 <br>
> 3 x 1 = 3 <br>
> 3 x 2 = 6 <br>
> 3 x 3 = 9 <br>
> 3 x 4 = 12 <br>
> 3 x 5 = 15 <br>
> 3 x 6 = 18 <br>
> 3 x 7 = 21 <br>
> 3 x 8 = 24 <br>
> 3 x 9 = 27 <br>
> 3 x 10 = 30 <br>

* 당신은 가변의(mutable) `<S>`를 사용할 수도 있다
* 앞의 예제는 다시 예로 들면, AtomicLong을 상태로 사용하여 재작성할 수 있다
* 각 순회마다 이것을 변환시킨다
* Example 12. Mutable state variant

```java
Flux<String> flux = Flux.generate(
    AtomicLong::new, 
    (state, sink) -> {
      long i = state.getAndIncrement(); 
      sink.next("3 x " + i + " = " + 3*i);
      if (i == 10) sink.complete();
      return state; 
    });
```

* This time, we generate a mutable object as the state.
* We mutate the state here.
* We return the same instance as the new state.

<br>

* "DB 연결이나 프로세스 마지막에 처리되어야 하는 다른 리소스"를 포함하는 상태의 경우, 
* consumer(소비자) lambda는 커넥션을 닫거나 프로세스 마지막에 처리되어야 하는 임의의 task를 처리할 수 있다

### 4.4.2. Asynchronous and Multi-threaded: create

* create는 더 발전된 Flux를 프로그래밍 방식으로 생성하는 형태이다
* 이것은 순회마다 여러 번의 emission에 적절하다.
* 심지어 멀티 스레드로부터 오는 emission도 가능하다

<br>

* 이것은 next, error, complete 메소드를 가진 FluxSink를 공개(expose)한다
* generate와는 달리, 이것은 상태 기반 sink가 없다
* 반면에, create는 멀티 스레드 이벤트를 콜백으로 trigger할 수 있다

<br>

* create는 이미 존재하는 API를 reactive world와 연결하는 데에 유용하다 - 예를 들면 listener 기반 비동기 API
* create가 비동기 API와 함께 사용될 수 있긴 하지만, 당신의 코드를 병렬화하거나 비동기로 만들어주지 않는다
* 만약 create lambda를 block시킬 경우, 당신은 스스로 deadlock이나 비슷한 부작용에 처하게 한다
* 심지어 subscribeOn을 사용해도, 긴 시간동안 create lambda를 blocking하는 것은 pipeline을 lock시킬 수 있다는 경고가 있다:
  * 예를 들면 sink.next(t) 무한 루프를 돌렸을 때
  * 이러한 요청들은 절대 수행되지 않는다. 루프문이 요청을 수행할 스레드에서 계속 돌고있기 때문이다
* subscribeOn(Scheduler, false)를 사용하라:
  * requestOnSeparateThread = false 는 스케줄러 스레드로 create를 수행하고, 
  * (여전히) 원래 스레드에서 요청을 수행함으로써 데이터를 흐르게 한다

<br>

* 당신이 linster 기반 API를 사용한다고 상상해보라. 이것은 데이터를 chunk로 처리하고 두 가지 이벤트를 가진다:
  * (1) 데이터의 청크가 준비되었다는 이벤트
  * (2) 처리가 완료되었다는 이벤트
* 이를 대표하는 MyEventListener interface이다

```java
interface MyEventListener<T> {
    void onDataChunk(List<T> chunk);
    void processComplete();
}
```

* 당신은 이것을 Flux와 연결하는 데에 create를 사용할 수 있다

```java
Flux<String> bridge = Flux.create(sink -> {
    myEventProcessor.register( 
      new MyEventListener<String>() { 

        public void onDataChunk(List<String> chunk) {
          for(String s : chunk) {
            sink.next(s); 
          }
        }

        public void processComplete() {
            sink.complete(); 
        }
    });
});
```

* Bridge to the MyEventListener API
* Each element in a chunk becomes an element in the Flux.
* The processComplete event is translated to onComplete.
* All of this is done asynchronously whenever the myEventProcessor executes

<br>

* 추가적으로, create는 비동기 API와 연결하고 backpressure를 관리할 수 있기 때문에, 
* 당신은 OverflowStrategy를 포함함으로써, backpressure-방식의 동작방식은 정제할 수 있다
  * IGNORE : 완전히 downstream의 backpressure 요청을 무시한다. downstream을 향한 큐가 가득찼을 때, 이것은 IllegalStateException 를 발생시킨다
  * ERROR : downstream이 더이상 버티지 못할 때, IllegalStateException 발생
  * DROP : 다운스트림이 받을 준비가 안 되었으면 그 시그널을 드랍시킴
  * LATEST : 다운스트림이 업스트림으로부터 오직 최근의 시그널만을 받도록 함
  * BUFFER (default) : 다운스트림이 받을 수 없을 경우 모든 시그널을 버퍼에 저장 (이것은 무한 버퍼링을 수행하고, OutOfMemoryError를 유발할 수 있음)

<br>

* Mono 또한 create generator를 가진다.
* Mono의 MonoSink는 한 번의 emission 후, 나머지 signal은 모두 drop한다

### 4.4.3. Asynchronous but single-threaded: push

* push는 generate와 create의 중간이다. 이것은 단일 producer에서 이벤트 처리에 적합하다.
* 이것은 create와 비슷하다 : push 또한 비동기가 될 수 있고, create에서 지원되는 overflow strategy를 사용하여 backpressure를 관리할 수 있다
* 하지만, push에서 각 시간마다(at a time) 오직 하나의 producing thread에서만 next, complete, error를 호출할 수 있다

```java
Flux<String> bridge = Flux.push(sink -> {
    myEventProcessor.register(
      new SingleThreadEventListener<String>() { 

        public void onDataChunk(List<String> chunk) {
          for(String s : chunk) {
            sink.next(s); 
          }
        }

        public void processComplete() {
            sink.complete(); 
        }

        public void processError(Throwable e) {
            sink.error(e); 
        }
    });
});
```

1. 단일 스레드 리스너 API와 연결한다
2. 이벤트들은 단일 리스너 스레드로부터 sink로 push된다
3. complete 이벤트는 같은 리스너 스레드로부터 생성된다
4. error 이벤트도 같은 리스너 스레드로부터 생성된다

* 궁금증 : Flux.push에 multi-thread를 사용하면 어떻게 될까?

> 테스트 결과, 일부 next signal들이 소실되었고, thread 수가 많아질수록 소실량이 더 커졌다. race condition에 의한 소실로 추측된다. <br>
> 예를 들어, 1..100의 이벤트가 발생해도 onNext에 도착한 시그널은 2, 3, 4, 5, 7, ...과 같이 사이사이 시그널들이 소실되었다 <br>
> multi-thread로 실행한다고 exception이 발생하거나 하지는 않았다. <br>

* 하이브리드 push/pull 모델
* 대부분의 리액터 오퍼레이터들은(create같은) 하이브리드 push/pull 모델을 따른다
* 이것이 의미하는 바는, 대부분의 프로세싱이 비동기지만 (push 접근방식을 제안하지만), 작은 pull 컴포넌트가 있다는 것이다: the request

<br>

* consumer 입장에서는, 처음 request되기 전까지 아무것도 emit하지 않는다는 의미로 데이터를 source로부터 pull한다.
> Iterator의 next()를 호출하여 pull하는 것과는 다르다는 의미인 듯 하다 <br>
* source 입장에서는 가능해지는대로 consumer에게 데이터를 push한다. 하지만 오직 요청된 양만큼만 보낸다.

<br>

* push()와 create() 둘다 onRequest 컨슈머를 허용한다.
* request 양을 관리하고 pending된 request가 존재할 때만 sink를 통해 데이터가 push되도록 하기 위함이다.

```java
Flux<String> bridge = Flux.create(sink -> {
    myMessageProcessor.register(
      new MyMessageListener<String>() {

        public void onMessage(List<String> messages) {
          for(String s : messages) {
            sink.next(s); 
          }
        }
    });
    sink.onRequest(n -> {
        List<String> messages = myMessageProcessor.getHistory(n); 
        for(String s : message) {
           sink.next(s); 
        }
    });
});
```

* push 또는 create 후 clean
* 두 콜백들, onDispose와 onCancel은 cancel이나 terminate를 수행한다
* onDispose는 Flux가 완료, error, cancel되었을 때 cleanup을 위해 사용된다.
* onCancel은 onDispose 전에 cancel을 위한 특정 액션을 수행하기 위해 사용된다.

```java
Flux<String> bridge = Flux.create(sink -> {
    sink.onRequest(n -> channel.poll(n))
        .onCancel(() -> channel.cancel()) 
        .onDispose(() -> channel.close())  
    });
```

### 4.4.4. Handle

`TODO`

## 4.5. Threading and Schedulers

`TODO`

## 4.6. Handling Errors

* 리액티브 스트림즈에서, 에러는 종료(terminal) 이벤트다. 에러는 발생하자마자 시퀀스를 멈추고, 오퍼레이터 체인 아래 쪽으로 전파되어 마지막 단계까지 간다
* 마지막 단계는 (당신이 정의한) subscriber의 onError method이다

<br>

* 이러한 에러들은 (여전히) 응용 레벨에서 처리되어야 한다.
  * 예를 들어, 에러 알림을 UI에 display하거나, REST endpoint에서 의미있는 에러 payload를 보내는 것이다
* 이러한 이유로, subscriber의 onError 메소드는 항상 정의되어야 한다

> 정의되지 않으면, onError는 UnsupportedOperationException을 throw한다. <br>
> 게다가 당신은 Exceptions.isErrorCallbackNotImplemented 메소드를 사용해 이것을 탐지하고 심사할 수 있다 <br>

* Reactor는 에러 핸들링 오퍼레이터로 체인의 중간에서 에러를 처리하는 방법의 대안 또한 제공한다
* 다음 예제는 어떻게 그렇게 하는지를 보인다

```java
Flux.just(1, 2, 0)
    .map(i -> "100 / " + i + " = " + (100 / i)) //this triggers an error with 0
    .onErrorReturn("Divided by zero :("); // error handling example
```

> 당신이 에러 핸들링 오퍼레이터를 배우기 전에, 리액티브 시퀀스에서 모든 에러는 종료 이벤트라는 것을 명심해야 한다 <br>
> 심지어 에러 핸들링 오퍼레이터가 사용되어도, 이것은 원래 시퀀스를 계속되게 하지 않는다 <br>
> 오히려, 이것은 onError 시그널을 새로운 시퀀스 (fallback)의 시작으로 변환한다 <br>
> 다른 말로, 이것은 종료된 시퀀스 업스트림을 대체한다 <br>

* 지금 우리는 에러 핸들링 방법을 하나하나 본다. 관련이 있는 경우, 우리는 try 패턴과 병렬적으로 만든다

### 4.6.1. Error Handling Operators

* 당신은 try-catch 블록으로 예외들을 처리하는 방법들과 더 친숙할 것이다
* 가장 명백하게, 그것들은 이런 것들을 포함합니다
  * catch하고 static default value를 리턴한다
  * catch하고 fallback 메소드로 대안 경로를 실행한다
  * catch하고 동적으로 fallback value를 연산한다
  * catch하고 BusinessException으로 wrap하여 re-throw한다
  * catch, log, and re-throw
  * finally 블록을 사용하여 리소스를 clean up한다 또는 Java 7 try-with-resource 생성자를 사용한다

<br>

* 리액터에는 이 모든 것들의 대응되는 것들이 에러 핸들링 오퍼레이터 형식으로 존재한다.
* 이 오퍼레이터들을 보기 전에, 우리는 먼저 리액티브 체인과 try-catch 블록 사이를 병렬적으로 비교해보려 한다

```java
Flux<String> s = Flux.range(1, 10)
    .map(v -> doSomethingDangerous(v)) 
    .map(v -> doSecondTransform(v)); 
s.subscribe(value -> System.out.println("RECEIVED " + value), 
            error -> System.err.println("CAUGHT " + error) 
);
```

* A transformation that can throw an exception is performed.
* If everything went well, a second transformation is performed.
* Each successfully transformed value is printed out.
* In case of an error, the sequence terminates and an error message is displayed.

<br>

* 앞의 예제는 개념적으로 다음 try-catch 블록과 같다

```java
try {
    for (int i = 1; i < 11; i++) {
        String v1 = doSomethingDangerous(i); 
        String v2 = doSecondTransform(v1); 
        System.out.println("RECEIVED " + v2);
    }
} catch (Throwable t) {
    System.err.println("CAUGHT " + t); 
}
```

* If an exception is thrown here…​
* …​the rest of the loop is skipped…​
* …​ and the execution goes straight to here.

* 병렬적인 비교를 해봤으므로, 다양한 에러 핸들링 경우들과 대응되는 오퍼레이터들을 확인해볼 수 있다



## 4.7. Processors

`생략`

## 5. Kotlin support

`TODO`

## 6. Testing

`TODO`

## 7. Debugging Reactor

`TODO`

## 7.4. Production-ready Global Debugging

* Reactor에는 분리된 Java Agent가 함꼐 제공된다
  * 이것은 코드를 계측(instrument)하고 디버깅 정보를 더해준다
  * 모든 operator call에서 stacktrace를 찍어낼(capture) 할 필요가 없다
* 이러한 동작은 traceback과 ㅂ슷하지만, 런타임 성능 오버헤드가 없다

<br>

* 사용을 위하여, 의존성 추가가 필요하다 (reactor-tools)
* 코드도 한 줄 필요하다

`ReactorDebugAgent.init()`

* 클래스들이 로딩될 때 계측하므로, main 메서드의 가장 앞에 높자
* 초기화를 eagerly 실행할 수 없다면 존재하는 클래스들을 re-process 할 수도 있다

```
ReactorDebugAgent.init();
ReactorDebugAgent.processExistingClasses();
```

* re-processing은 시간이 좀 걸린다. 로딩된 모든 클래스들을 순회하고 transform 해야하기 때문에.
* 오직 일부 호출구역(call-sites)이 계측되지 않는 경우에만 사용하자.

### 7.4.1. Limitations

* ReactorDebugAgent는 Java Agent로 구현되었고, self-attach를 위해 ByteBuddy를 사용한다
  * self-attach란, debug agent가 debugging을 위해 응용 프로그램에 붙는 것을 의미한다
* self-attach는 몇몇 JVM에서 동작하지 않는다. 자세한 내용은 BuyteBuddy 문서를 참조하라.

|Keyword|Description|
|-------|-----------|
|ByteBuddy||

## 7.5. Logging a Sequence
