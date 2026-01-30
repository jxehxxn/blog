---
layout: post
title: "Network & App Hacking Mastery: Week 11 - Automating the Attack, Building Your Toolkit"
---

이제 여러분은 네트워크도 장악했고(mitmproxy), 앱 내부도 장악했습니다(Frida).
하지만 진정한 고수는 도구를 섞어서 씁니다.

앱이 통신할 때 **E2EE (End-to-End Encryption)**를 적용해서, HTTPS 안에서도 독자적인 암호화를 쓴다고 가정해 봅시다.
mitmproxy로 패킷을 까봐도 암호화된 쓰레기 값만 보입니다. SSL Pinning을 뚫어도 소용이 없습니다.
어떻게 해야 할까요?

---

## 🤝 Combining mitmproxy + Frida

해답은 **공조(Collaboration)**입니다.

1.  **Frida**는 앱 내부의 암호화 함수(`AES.encrypt`)를 후킹합니다.
    *   암호화되기 **직전의 평문(Plaintext)**을 훔칩니다.
    *   이 평문을 mitmproxy에게 몰래 보내줍니다.
2.  **mitmproxy**는 받은 평문을 화면에 보여줍니다.
    *   우리가 평문을 수정해서 다시 보내면?
    *   Frida가 그걸 받아서 다시 암호화해서 서버로 보냅니다.

이렇게 하면 **"투명한 분석 파이프라인"**이 완성됩니다. 암호화가 있든 없든 우리는 평문을 보고 조작할 수 있습니다.

---

## 🛠️ Lab: The Decryption Proxy (Conceptual)

이 기술은 고급 기술이므로 개념적으로 설계해 봅니다.

### 1. Frida Script
```javascript
// AES 암호화 함수 후킹
var AES = Java.use("com.example.security.AES");
AES.encrypt.implementation = function(plaintext) {
    // 1. 평문을 훔쳐서 로컬 서버(mitmproxy 등)로 전송
    send({type: "log", payload: plaintext});
    
    // 2. 원래 암호화 진행
    return this.encrypt(plaintext);
};
```

### 2. Python Handler (PC)
Frida가 `send()`로 보낸 메시지를 파이썬 스크립트가 받아서 파일로 저장하거나 화면에 출력합니다.

---

## 🤖 Automating Pentesting

단순 분석뿐만 아니라 공격도 자동화할 수 있습니다.
예를 들어 **로그인 무차별 대입(Brute Force)**을 한다고 칩시다.
네트워크로만 쏘면 캡차(CAPTCHA)나 암호화에 막힐 수 있습니다.

Frida로 **UI 버튼 클릭 이벤트**를 호출해버리면 됩니다.
```javascript
// 1초마다 아이디/비번을 입력하고 '로그인' 버튼을 클릭하는 루프
setInterval(function() {
    loginFunc("admin", generateNextPassword());
}, 1000);
```
이것이 바로 **Runtime Automator**입니다.

---

## 📝 11주차 과제: 툴 설계

여러분이 만약 "모바일 게임 매크로"나 "자동 취약점 스캐너"를 만든다면,
Frida와 mitmproxy를 어떻게 조합할지 아키텍처를 그려보세요. (글로 서술)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 앱이 HTTPS 외에 자체적으로 데이터를 한 번 더 암호화하여 전송하는 방식을 무엇이라 하나요? (E2EE)
2.  **Q2.** Frida 스크립트 내에서 외부(PC의 파이썬 스크립트 등)로 데이터를 전송하기 위해 사용하는 함수는? (`s...`)
3.  **Q3.** 네트워크 패킷 레벨이 아니라, 앱 내부 함수의 입력값을 조작하여 암호화를 우회하는 접근 방식의 장점은 무엇인가요? (키 값을 몰라도 평문을 볼 수 있음)

---

다음 주는 대망의 **마지막 주**입니다.
지금까지 배운 모든 기술을 총동원하는 **Capstone Project**가 기다리고 있습니다.
가상의 타겟 앱을 상대로 Rooting 탐지를 뚫고, Pinning을 우회하고, 숨겨진 Flag를 찾아내세요.

**Build your weapon.**
