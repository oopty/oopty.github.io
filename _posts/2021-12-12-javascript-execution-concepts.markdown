---
layout: post
title:  "자바스크립트 동작 원리에대한 개념 정리"
date:   2021-12-12 00:23:45 +0900
categories: javascript
---
## 시작하며
이번에는 자바 런타임에서 바이트코드를 실행할 수 있는 JVM(Java Virtual Machine)과 Javascript 엔진인 V8에 대해 비교하며 살펴보려고 합니다. 개발하다 보면 많이 듣는 용어고 의미정도는 어렴풋이 알고 있긴 하지만 설명을 하라고 하면 못할 거 같습니다.. 😓 그래서 이번엔 V8과 JVM에 대해 정리하고 관련된 용어(`Node.js`, `ECMAScript`, `JDK`, `GC`)도 공부해보려고 합니다. 잘못된 정보나 제 말에 억측이 있어 짚어주신다면 고맙겠습니다~ 😊

## V8 엔진에 대해서

V8 개발자 사이트([v8.dev](https://v8.dev/))에 접속하면 아래와 같이 V8을 말하고 있다.
> V8 is Google’s open source high-performance JavaScript and WebAssembly engine, written in C++. It is used in Chrome and in Node.js, among others. It implements ECMAScript and WebAssembly, and runs on Windows 7 or later, macOS 10.12+, and Linux systems that use x64, IA-32, ARM, or MIPS processors. V8 can run standalone, or can be embedded into any C++ application.

정리하면 `C++`로 구현된 `Javascript와` `WebAssembly` 엔진이고 ECMAScript의 구현체이다. 또한 여러 플랫폼 위에서 standalone 방식으로 실행될 수 잇고 C++ 임베디드 환경에서도 사용 가능하다. Javascript는 익숙한데 WebAssembly는 무엇일까? WebAssembly는 스택기반 가상환경에서 실행할 수 있는 바이너리 코드 포맷이다. 웹의 브라우저나 서버에 배포할 수 있도록 포터블하게 디자인 되어있다. 쉽게 말하면, C나 C++로 프로그래밍한 결과를 브라우저에서 실행시켜주는 기술이다. 장점으로는 빠르고 효과적인 실행이 있다. 네이티브 어플리케이션의 속도를 목표로 한다고 한다. 또한 웹 플랫폼의 일부분으로서 `Javascript`에서 실행될 수 있고 역으로 `Javascript`를 실행할 수 도 있다. 더 알고 싶다면 [webassembly](https://webassembly.org/)를 참고하자. 어쨋든 V8엔진은 `Javascript`뿐만 아니라 `WebAssembly`도 포함한 엔진이다.

ECMAScript는 Ecma International(국제 정보 통신 시스템 국제화 기구)이 ECMA-262 기술 규격에 따라 정의하고 있는 표준화된 스크립트 프로그래밍 언어이다. 자바스크립트를 표준화하기 위해 만들어 졌고 액션스크립나 J스크립트 등 다른 구현체도 있다. `Node.js`같이 서버 응용 프로그램에도 쓰이고 있다. 쉽게 말하면 자바스크립트를 표준화시킨 규격이다.

## V8 엔진 구조

프런트엔드 코드를 작성하면서 알 필요는 없지만 항상 궁금했다. C나 C++은 학교 교수님이 어떻게 어셈블리어로 바뀌고 실행이 되는지 자세하진 않지만 설명을 들었고 그 만큼 코드가 어떻게 동작할지 예상이 되었다. V8 엔진에 대해서도 대략적으로나마 알아보자

![v8엔진](/assets/images/v8-engine.png)

V8은 내연 기관에 들어가는 8기통 엔진에서 유래되었다고 한다. 각 크랭크실에 가스를 분사한 후 점화를 일으켜 피스톤이 운동에너지를 전달한다. 과열된 엔진은 그릴에서 들어오는 바람이나 팬으로 냉각한다. 자바스크립트 엔진인 V8에서도 비슷한 내부 로직이 보인다. V8의 parser는 자바스크립트 소스를 읽어 들여 AST(추상 문법 트리)로 만든다. 그리고 Interpreter Ignition으로 byteCode를 만들어 실행할 수 있게 점화한다. 어떤 코드가 유난히 많이 실행이 되면 TurboFan이 코드 최적화를 시켜 엔진의 속도를 높인다. 더 자세히 알면 좋지만 byteCode나 TurBoFan이 어떻게 최적화 하는지는 나중에 찾아보기로 한다.

## 가비지 컬렉터(GC)

C와 같은 저수준 언어에서는 프로그래머가 명시적으로 메모리의 할당과 해제를 수행하지만 자바스크립트 같은 고수준 언어는 객체가 생성되었을 때 자동으로 메모리를 할당하고 필요없을 때 해제한다. 하지만 이런 메모리 관리가 개발자에게 메모리 관리에 대한 고민을 완전히 없애고 신경쓰지 않아도 된다는 것은 아니다. 메모리의 생명주기는 언어에 관계없이 대부분 아래와 같다.

1. 메모리 생성
2. 사용한다.
3. 필요없어지면 해제한다.

3번에서 대부분의 문제가 발생한다. 언제 필요없어지는지 알수가 없기 때문이다. 어떤 메모리가 여전히 필요한지 아닌지를 판단하는 것은 [비결정적 문제](https://en.wikipedia.org/wiki/Decidability_%28logic%29)이기 때문이다. 따라서 가비지 컬렉터(GC)는 제한적인 해결책을 제시한다.

- 참조-세기(Reference-counting) 가비지 컬렉션
 단순한 알고리즘이다. ```더 이상 필요없는 오브젝트```를 ```어떤 다른 오브젝트도 참조하지 않는 오브젝트```라고 정의한다. 다시말하면 자신 또는 자신의 속성을 참조하는 변수가 없는 오브젝트는 삭제하는 것이다.
 하지만 아래와 같은 순환 참조에서는 스코프를 벗어나 객체가 메모리에서 회수되어야 하지만 회수되지 않는다. 참조-세기 가비지 컬렉터에서 메모리 누수의 흔한 원인이다.

 ```js
function f() {
    var x = {};
    var y = {};
    x.a = y;
    y.a = x;

    return "oopty!";
}

f();
 ```

 자기 자신의 속성이 자신을 참조하는 경우도 순환참조라고 볼 수 있다.
- 표시하고-쓸기(Mark-and_sweep) 알고리즘  
  

 이 알고리즘은 ```더 이상 필요없는 오브젝트```를 ```닿을 수 없는 오브젝트```로 정의한다.
 ``roots``라는 오브젝트 집합을 가지고 있고 주기적으로 가비지 컬렉터는 ``roots``에서 시작하여 참조할 수 있는 오브젝트를 닿을 수 있는 오브젝트라고 표시한다. 그리고 닿을 수 없는 오브젝트들에 대해 가비지 컬렉터를 수행한다.  

 이는 참조-세기 알고리즘보다 효율적이다. 즉 더 많은 오브젝트의 메모리를 해제한다. 왜냐하면 ```참조하지 않는 오브젝트```는 모두 ```닿을 수 잇는 오브젝트```이지만 역은 성립하지 않기 때문이다. 반례로 순환 참조하는 오브젝트를 들수있다. 2012년 기준으로 모든 최신 브라우저는 표시하고-쓸기(Mark-and-sweep) 알고리즘을 사용한다고 한다.

- 수동 메모리 해제

 2019년 기준으로 자바스크립트는 메모리 해제를 명시적으로 할 수 없다.

- 힙 메모리 사용량 늘리기

```
node --max-old-space-size=6000 index.js
```

- 가비지 수집기 디버깅

```
node --expose-gc --inspect index.js
```

## 이벤트 루프
자바스크립트는 '단일 스레드'기반의 언어다. 하지만 브라우저를 사용해 보면 렌더링을 하는 동시에 HTTP request도 보내고 setTimeout으로 콜백을 전달하기도 한다. 이러한 동작은 단일 스레드에서 어떻게 수행하는 것일까? 즉, 자바스크립트의 동시성(Concurrency)는 어떻게 동작하는 것일까?

![이벤트 루프 고수준 관점](/assets/images/highLevelEventLoopView.jpg)

이벤트 루프는 V8엔진에 있는 것이 아니라 비동기 I/O를 지원하는 멀티 플랫폼 C라이브러리인 LibUV에 있다. 자바스크립트의 규격인 ECMAScript에는 이벤트 루프나 비동기에 관련된 언급이 없다고 한다. libUV가 있기 때문에 자바스크립트에서 동시성을 사용할 수 있는 것이다. 

![이벤트 루프 저수준 관점](/assets/images/lowLevelEventLoopView.png)

총 7개의 페이즈가 있고 각 페이즈마다 특정 역할이 있다. 각 페이즈는 큐를 가지고 있고 자바스크립트는 idle과 prepare을 제외하고 실행될 수 있다.

### Timer Phase  
이벤트 루프의 시작이고 타이머들의 콜백을 min heap에 가지고 있다가 시간이 지나면 해당 타이머를 큐에 넣고 실행한다.

### Pending i/o callback phase  
`pending_queue`에 들어가 있는 콜백을 실행한다. `poll phase`에서 pending된 i/o 콜백을 실행하는 거 같다.

### Idle, Prepare pahse  
이름은 Idle Phase이지만 이 페이즈는 매 tick마다 실행한다. Node.js의 내부관리를 위해 사용하는 것이다.

### Poll phase  
새로운 수신 커넥션과 데이터 읽기/쓰기 등 대부분의 콜백을 실행한다. `watch_queue`가 비어있지 않다면, 큐가 비거나 시스템 최대 실행 한도에 다다를때까지 동기적으로 콜백을 실행, 비어있다면 `Check phase` 로 넘어가거나 약간의 대기시간을 가지게 되고 타이머 min heap에서 하나를 꺼내고 타이머가 실행 가능한 상태면 타이머의 딜레이 시간만큼 대기한다.

### Check phase  
`setImmediate()` API의 콜백을 큐가 비거나 실행 한도가 초과될 때까지 실행한다.

### Close Callback  
커넥션을 닫거나 종료하는 콜백을 실행한다.

### nextTickQueue & microTaskQueue  
다음 페이즈로 넘어가기 전에 두 큐에 있는 콜백을 높은 우선순위로 실행한다. `nextTickQueue`가 `microTaskQueue`보다 높은 우선순위를 가지고 있다.


## 마치면서
프로그래밍을 하면서 그냥 지나쳤던 자바스크립트의 동작 원리에 대해 간단히 정리해 보았습니다. 대략적인 개념 정도로 정리했고 자세한 내용은 포스팅 하나하나로 정리해야 할 거 같습니다. 특히 이벤트 루프관련된 내용이 자바스크립트 실행 순서와 관련된 내용이라서 꼭 한번 정리하겠습니다.

## 참고자료
- [Evans Library: 로우레벨로 살펴보는 Node.js 이벤트 루프](https://evan-moon.github.io/2019/08/01/nodejs-event-loop-workflow/)
- [NHN Cloud MeetUp: 자바크립트와 이벤트 루프](https://meetup.toast.com/posts/89)
- [MDN Web Docs: 동시성 모델과 이벤트 루프](https://developer.mozilla.org/ko/docs/Web/JavaScript/EventLoop)
- [MDN Web Docs: 자바스크립트의 메모리관](https://developer.mozilla.org/ko/docs/Web/JavaScript/Memory_Management)