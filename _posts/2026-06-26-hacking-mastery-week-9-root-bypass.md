---
layout: post
title: "Network & App Hacking Mastery: Week 9 - Prison Break, Bypassing Root & Jailbreak Detection"
---

보안이 적용된 금융 앱이나 게임을 켜면 **"루팅된 단말기에서는 사용할 수 없습니다"**라는 메시지와 함께 꺼집니다.
해커의 분석 도구(Frida, mitmproxy)가 돌아가는 환경을 원천 봉쇄하려는 것이죠.

오늘은 이 탐지 로직을 우회하여, 보안의 눈을 피해 앱을 실행시키는 **Prison Break**를 감행합니다.

---

## 🐱 Cat and Mouse Game

루팅 탐지는 숨바꼭질입니다.
앱은 다음과 같은 증거를 찾습니다.

1.  **파일 검사:** `/system/xbin/su`, `/system/app/Superuser.apk` 파일이 있나?
2.  **패키지 검사:** `com.topjohnwu.magisk` 같은 패키지가 설치됐나?
3.  **명령어 실행:** `su` 명령어가 먹히나?
4.  **속성 검사:** `ro.build.tags`가 `test-keys`인가?

우리는 이 질문들에 대해 **거짓말**을 해야 합니다.

---

## 🛡️ Strategies: 거짓말의 기술

### 1. Java Hooking (File.exists)
앱이 `File` 클래스의 `exists()` 메소드를 호출해서 `su` 파일을 찾으려 할 때 후킹합니다.

```javascript
var File = Java.use("java.io.File");
File.exists.implementation = function() {
    var path = this.getAbsolutePath();
    if (path.indexOf("/su") > -1) {
        console.log("Root detection blocked: " + path);
        return false; // "그런 파일 없는데요?"
    }
    return this.exists.call(this); // 다른 파일은 정상적으로 처리
};
```

### 2. Native Hooking (fopen, access)
더 강력한 앱은 C언어 레벨에서 `fopen`이나 `access` 함수로 파일을 확인합니다.
지난주(8주차)에 배운 네이티브 후킹으로 파일 경로를 검사하고, 루팅 관련 파일이면 "파일 없음(ENOENT)" 에러를 리턴하도록 조작해야 합니다.

---

## 🛠️ Tool: Objection

매번 스크립트를 짜는 건 귀찮습니다. **Objection**이라는 도구는 미리 짜여진 수많은 우회 스크립트를 내장하고 있습니다.

```bash
# 1. 앱에 Objection 붙이기
objection -g com.example.app explore

# 2. 루팅 탐지 우회 명령 실행
android root disable
```
이 한 줄이면 수십 가지의 탐지 로직을 자동으로 무력화시킵니다. (물론 만능은 아닙니다. 실패하면 직접 스크립트를 짜야 합니다.)

---

## 📝 9주차 과제: RootBeer Bypass

**RootBeer**는 안드로이드 루팅 탐지 라이브러리 데모 앱입니다.
이 앱을 설치하고 실행하면 화면이 빨갛게 변하며 루팅됨을 알립니다.
Frida 스크립트(또는 Objection)를 사용하여 모든 체크 항목을 **초록색(Not Rooted)**으로 만드세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 앱이 루팅 여부를 확인하기 위해 가장 흔하게 검사하는 바이너리 파일의 이름은 무엇인가요?
2.  **Q2.** 루팅 탐지 우회를 위해 파일 존재 여부를 확인하는 자바 메소드(`exists`)를 후킹하여, 특정 경로일 때 어떤 값을 리턴해야 하나요? (`true` vs `false`)
3.  **Q3.** Frida 기반으로 만들어진 런타임 모바일 탐색 도구로, `android root disable` 같은 편리한 명령어를 제공하는 툴의 이름은?

---

다음 주는 네트워크 해킹의 끝판왕, **SSL Pinning 우회**입니다.
4주차에는 이론만 배웠지만, 이제 Frida라는 무기가 생겼으니 실제로 뚫어볼 차례입니다.

**Hide the keys.**
