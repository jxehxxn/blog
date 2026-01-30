---
layout: post
title: "Network & App Hacking Mastery: Week 8 - Native Hooking, Diving into C/C++"
---

앱이 느려지거나, 정말 중요한 보안 로직(암호화, 무결성 검사)을 짤 때 개발자들은 Java를 버리고 **C/C++**을 선택합니다.
안드로이드에서는 이것을 **JNI (Java Native Interface)**라고 부르며, `.so` (Shared Object) 라이브러리 파일로 존재합니다.

Java 레이어만 봐서는 절대 풀 수 없는 수수께끼가 여기에 숨어 있습니다.
오늘은 더 깊은 심연, **Native World**로 다이빙합니다.

---

## 💾 Memory Hacking

Native 후킹은 메소드 이름이 아니라 **메모리 주소(Address)**를 다룹니다.
물론 `Export`된 함수(외부에서 부를 수 있는 함수)는 이름으로 찾을 수 있습니다.

### libc 후킹 (시스템 콜 가로채기)

모든 앱은 결국 운영체제의 기능을 씁니다. 파일을 열거나(`open`), 네트워크를 쓰거나(`connect`), 글자를 비교하거나(`strcmp`).
이런 표준 라이브러리 함수를 후킹하면 앱이 무슨 짓을 하는지 감시할 수 있습니다.

```javascript
// 'open' 함수의 주소를 찾습니다.
var openPtr = Module.findExportByName(null, "open");

Interceptor.attach(openPtr, {
    onEnter: function(args) {
        // 첫 번째 인자(args[0])는 파일 경로 문자열의 포인터입니다.
        // 포인터가 가리키는 곳의 문자열을 읽어옵니다.
        var path = Memory.readUtf8String(args[0]);
        console.log("File Open Attempt: " + path);
    },
    onLeave: function(retval) {
        // 리턴값은 파일 디스크립터(fd)입니다.
    }
});
```

---

## 🛠️ Lab: Spy on Files

앱이 시작될 때 어떤 설정 파일을 몰래 읽는지, 혹은 루팅 탐지를 위해 어떤 시스템 파일을 찌르는지 감시해 봅시다.

1.  위의 `open` 후킹 스크립트를 준비합니다.
2.  타겟 앱을 실행하면서 스크립트를 주입합니다.
3.  로그를 봅니다.
    *   `/data/user/0/com.example/shared_prefs/secret.xml`
    *   `/system/xbin/su` (어? 이거 루팅 검사하는 거네?)

이렇게 앱의 **행동 패턴**을 분석할 수 있습니다.

---

## 📝 8주차 과제: strcmp 후킹

`strcmp` (문자열 비교 함수)는 비밀번호 비교 로직의 단골 손님입니다.
`strcmp`를 후킹해서, 앱이 비교하는 두 문자열을 모두 로그로 출력하는 스크립트를 짜보세요.
운이 좋다면 사용자가 입력한 값과, **하드코딩된 정답**이 나란히 출력될 수도 있습니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 안드로이드에서 Java 코드와 C/C++ 네이티브 코드 사이를 연결해주는 인터페이스를 약어로 무엇이라 하나요?
2.  **Q2.** Frida에서 네이티브 함수의 주소를 찾기 위해 사용하는 API는? (`Module.find...`)
3.  **Q3.** 네이티브 후킹 시 함수 진입(`onEnter`) 단계에서 인자값(포인터)이 가리키는 메모리 내용을 문자열로 읽어오는 함수는? (`Memory.read...`)

---

다음 주는 이 기술을 활용해 **탈옥/루팅 탐지 우회(Bypassing)**를 실습합니다.
"너 루팅된 폰이지? 앱 꺼!"라고 협박하는 보안 모듈을,
"아닌데요? 순정 폰인데요?"라고 천연덕스럽게 거짓말하도록 만들어 봅시다.

**Read the memory.**
