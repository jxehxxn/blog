---
layout: post
title: "Network & App Hacking Mastery: Week 4 - The Fortress, Certificate Pinning & Client Checks"
---

mitmproxy를 켜고, CA 인증서도 잘 설치했습니다. 브라우저에서는 네이버도 구글도 잘 됩니다.
그런데 특정 앱(금융 앱, 게임 등)을 켜니까... **"네트워크 연결이 원활하지 않습니다"**라며 먹통이 됩니다.

축하합니다. 여러분은 방어 기법의 최전선, **Certificate Pinning(인증서 고정)**을 마주쳤습니다.

---

## 🏰 The Problem: 신뢰의 거부

우리는 1주차에 "내 가짜 신분증(CA)을 믿어줘"라고 폰 설정에 등록했습니다.
대부분의 앱은 운영체제(OS)가 믿는 신분증을 같이 믿습니다.
하지만 보안이 중요한 앱은 이렇게 말합니다.

> **App:** "운영체제가 널 믿든 말든 상관 안 해. 난 우리 서버가 가진 **진짜 신분증의 지문(Pin)**을 알고 있어. 그거랑 똑같은지 비교해볼 거야."

이것이 **Pinning**입니다. 우리가 중간에서 가짜 인증서를 내미니까, 미리 알고 있던 지문과 달라서 연결을 끊어버리는 겁니다.

---

## 🔒 Theory: 어떻게 작동하나?

앱 내부(소스 코드)에 서버 인증서의 해시값(지문)이 하드코딩되어 있습니다.
TLS 연결이 맺어질 때, 서버가 보내준 인증서의 해시값을 계산해서 하드코딩된 값과 비교합니다.

*   **일치:** "진짜 우리 서버네." -> 통신 시작
*   **불일치:** "너 누구야? MITM 공격이니?" -> 연결 종료 (`SSLHandshakeException`)

---

## ⚔️ Bypassing Strategies (뚫는 법)

이 벽을 어떻게 넘을까요?

1.  **Old Way (Re-packaging):**
    *   앱(APK)을 뜯어서(Decompile), 소스 코드에서 Pinning 검사하는 코드를 찾습니다.
    *   코드를 지우고 다시 조립(Recompile)합니다.
    *   **단점:** 앱이 위변조를 감지(Integrity Check)해서 실행되지 않을 확률이 높습니다. 너무 어렵습니다.

2.  **New Way (Runtime Hooking):**
    *   앱을 뜯지 않습니다. 실행 중에 **메모리**를 조작합니다.
    *   "지문 비교하는 함수"를 찾습니다.
    *   그 함수가 무조건 `true`(일치함)를 리턴하도록 조작합니다.
    *   이것을 위해 다음 주부터 **Frida**를 배울 것입니다.

---

## 🛠️ Lab: Detecting Pinning (피닝 감지하기)

아직 Frida를 배우기 전이니, 오늘은 mitmproxy 로그를 보며 피닝이 걸려있는지 확인하는 법을 익힙니다.

1.  mitmproxy를 켭니다.
2.  대상 앱을 실행합니다.
3.  앱에서 "연결 오류"가 뜹니다.
4.  mitmproxy 로그(Event Log)를 확인합니다.
    *   `Client Handshake Failed`
    *   `The client may not trust the proxy's certificate`
    *   `TLS alert: bad certificate`

이런 로그가 보이면 99% 피닝입니다.

---

## 📝 4주차 과제: Objection & OkHttp 조사

다음 주부터 사용할 도구와 라이브러리에 대해 예습해 오세요.

1.  **Objection:** Frida를 쉽게 쓰게 해주는 툴입니다. 어떤 기능이 있는지 찾아보세요.
2.  **OkHttp:** 안드로이드에서 가장 많이 쓰는 통신 라이브러리입니다. 여기서 `CertificatePinner`라는 클래스가 무슨 일을 하는지 검색해 보세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 앱이 운영체제의 신뢰 저장소를 무시하고, 미리 지정된 인증서만 신뢰하는 보안 기법을 무엇이라 하나요?
2.  **Q2.** Certificate Pinning이 적용된 앱을 프록시로 분석하려 할 때 발생하는 대표적인 에러 메시지 키워드는? (TLS, Handshake 등)
3.  **Q3.** Pinning을 우회하기 위해 앱을 변조(Re-packaging)하는 것보다, 실행 중에 메모리를 조작하는 방식이 선호되는 이유는 무엇 때문인가요? (앱의 무결성 검증 때문)

---

자, 네트워크(Part 1)는 여기까지입니다.
이제 우리는 더 깊은 곳, **앱의 메모리(Part 2)**로 들어갑니다.
**Frida**라는 강력한 메스를 들고, 런타임의 세계를 해부하러 갑시다.

**Prepare the injection.**
