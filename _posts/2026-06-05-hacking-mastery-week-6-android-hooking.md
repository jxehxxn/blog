---
layout: post
title: "Network & App Hacking Mastery: Week 6 - Android Hooking, Controlling the Java Layer"
---

안드로이드 앱은 대부분 **Java**나 **Kotlin**으로 짜여 있습니다.
이 말은 즉, Java 가상머신(Dalvik/ART) 위에서 돌아간다는 뜻입니다.

Frida는 이 가상머신에 접근할 수 있는 강력한 API인 `Java.perform()`을 제공합니다.
오늘은 이것을 이용해 앱의 로직을 내 맘대로 바꿔보겠습니다.

---

## ☕ The Java Layer

안드로이드 앱의 모든 클래스와 메소드는 메모리에 로드되어 있습니다.
우리는 Frida를 통해 특정 클래스를 찾고(`Java.use`), 그 클래스의 메소드를 덮어쓸(`implementation`) 수 있습니다.

### 핵심 문법

```javascript
Java.perform(function() {
    // 1. 클래스 찾기
    var MyClass = Java.use("com.example.app.MainActivity");

    // 2. 메소드 덮어쓰기
    MyClass.isPremiumUser.implementation = function() {
        console.log("Hooked! Returning true unconditionally.");
        return true; // 무조건 프리미엄 유저라고 뻥치기
    };
});
```

---

## 🛠️ Lab: Unlocking the App (CrackMe)

간단한 **CrackMe** 앱(비밀번호를 맞춰야 넘어가는 앱)을 대상으로 실습해 봅시다.

### 상황
앱에 비밀번호 입력창이 있고, 틀리면 "Fail", 맞으면 "Success"가 뜹니다.
우리는 정답을 모릅니다.

### 분석
코드를 분석해보니(jadx 등 사용) `checkPassword(String input)`라는 함수가 있고, 이게 `boolean`을 리턴합니다.

### 해킹 (Hooking)
```javascript
Java.perform(function() {
    var MainActivity = Java.use("com.example.crackme.MainActivity");
    
    // checkPassword 함수를 가로챕니다.
    MainActivity.checkPassword.implementation = function(input) {
        console.log("사용자가 입력한 값: " + input);
        
        // 원래 검증 로직은 무시하고, 무조건 true(정답)를 리턴합니다.
        return true; 
    };
});
```

이제 앱에 "1234"를 넣든 "abcd"를 넣든, 앱은 "Success"를 외칠 것입니다. 이것이 **Runtime Hooking**의 위력입니다.

---

## 📝 6주차 과제: UI 조작하기

단순히 리턴값만 바꾸는 게 아니라, UI도 바꿔봅시다.
앱 화면에 있는 `TextView`의 내용을 "Hacked by Frida"로 바꾸는 스크립트를 작성해 보세요.

**힌트:**
1.  `TextView` 클래스를 `Java.use`로 가져옵니다.
2.  `setText` 메소드를 후킹합니다.
3.  원래 `setText`를 호출(`this.setText.call(...)`)하되, 인자값을 바꿔치기해서 호출합니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Frida에서 안드로이드 Java 레이어에 접근하기 위해 가장 먼저 호출해야 하는 함수는 무엇인가요? (`Java....`)
2.  **Q2.** 특정 클래스의 원본 메소드 로직을 새로운 자바스크립트 함수로 대체하기 위해 사용하는 속성(Property) 이름은? (`.im...`)
3.  **Q3.** 같은 이름의 메소드가 여러 개 있을 때(인자가 다른 경우), 특정 메소드를 지정하기 위해 사용하는 키워드는? (`.over...`)

---

다음 주는 **iOS**입니다. (맥이 없어도 괜찮습니다. 이론은 알아야 하니까요.)
안드로이드와는 다른 Objective-C의 세계, 그리고 스위즐링(Swizzling)의 마법을 엿보겠습니다.

**Overload and Override.**
