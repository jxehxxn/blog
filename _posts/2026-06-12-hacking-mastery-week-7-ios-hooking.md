---
layout: post
title: "Network & App Hacking Mastery: Week 7 - iOS Hooking, Objective-C & Swift Magic"
---

모바일 해킹의 양대 산맥, **iOS**입니다.
아이폰은 안드로이드보다 보안이 폐쇄적이지만, Frida에게는 똑같은 놀이터일 뿐입니다.
iOS 앱은 주로 **Objective-C**나 **Swift**로 작성됩니다. 언어는 다르지만 후킹의 원리는 같습니다.

---

## 🍏 The Other World: Objective-C Runtime

Objective-C는 굉장히 동적인 언어입니다.
실행 중에 메소드를 바꿔치기하는 **Method Swizzling**이라는 기법이 언어 자체적으로 지원될 정도니까요.
Frida는 이 런타임에 접근하여 `ObjC` API를 통해 후킹을 수행합니다.

### 핵심 문법

```javascript
// 클래스 찾기 (안드로이드의 Java.use와 비슷)
var className = "User";
var funcName = "- isPremium"; // -는 인스턴스 메소드, +는 클래스 메소드

// 후킹 포인트(주소) 찾기
var hook = ObjC.classes[className][funcName];

// 인터셉터 부착
Interceptor.attach(hook.implementation, {
    onEnter: function(args) {
        console.log("isPremium called!");
    },
    onLeave: function(retval) {
        console.log("Original Return: " + retval);
        retval.replace(1); // 1은 True를 의미. 리턴값 조작!
    }
});
```

---

## 🔍 Swift?

Swift는 정적인 언어라 Objective-C보다 후킹이 까다롭습니다.
하지만 대부분의 Swift 앱은 Objective-C 런타임과 호환되거나 다리(Bridge)를 걸치고 있습니다.
순수 Swift 함수라도 **이름 맹글링(Name Mangling)**을 풀어서 주소를 찾아내면 `Interceptor.attach`로 후킹할 수 있습니다.

---

## 🛠️ Lab: Simulator Practice (Conceptual)

맥(Mac)이 없는 분들을 위해 개념적으로 설명합니다.
탈옥된 아이폰이나 시뮬레이터가 있다면 직접 해보셔도 좋습니다.

1.  **타겟:** 프리미엄 기능을 체크하는 `- [User isPremium]` 메소드.
2.  **액션:** 앱이 실행되자마자 이 메소드의 리턴값을 `1`로 고정.
3.  **결과:** 결제하지 않아도 프리미엄 메뉴가 열림.

안드로이드의 `checkPassword` 실습과 논리적으로 완벽히 동일합니다. 문법만 다를 뿐입니다.

---

## 📝 7주차 과제: 문법 비교

안드로이드(`Java.perform`)와 iOS(`ObjC.classes`) 후킹 스크립트의 차이점을 표로 정리해 보세요.
*   클래스를 찾는 방법
*   메소드를 지정하는 방법
*   인자값과 리턴값을 건드리는 방법

---

## ✅ Self-Assessment Quiz

1.  **Q1.** iOS 후킹 시 Frida에서 Objective-C 런타임에 접근하기 위해 사용하는 전역 객체의 이름은?
2.  **Q2.** Objective-C 메소드 이름 앞에 붙는 `-`와 `+`는 각각 무엇을 의미하나요?
3.  **Q3.** 이미 생성되어 메모리 힙(Heap)에 존재하는 객체 인스턴스를 찾아내기 위해 사용하는 Frida 함수는? (`ObjC.ch...`)

---

다음 주는 언어의 장벽을 넘어, **Native Hooking**으로 들어갑니다.
Java도 아니고 ObjC도 아닌, **C/C++** 레벨의 메모리를 직접 건드립니다.
게임 해킹이나 보안 솔루션 우회에 필수적인 기술입니다.

**Swizzle the methods.**
