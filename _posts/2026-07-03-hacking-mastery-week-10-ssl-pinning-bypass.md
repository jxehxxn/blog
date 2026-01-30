---
layout: post
title: "Network & App Hacking Mastery: Week 10 - The Ultimate Duel, Bypassing SSL Pinning"
---

4주차를 기억하시나요?
mitmproxy를 켰더니 앱이 연결을 거부했던 그 좌절감을요.
**Certificate Pinning**은 강력하지만, 결국 앱 내부의 **코드**일 뿐입니다.
그리고 우리는 이제 실행 중인 코드를 맘대로 바꿀 수 있는 **Frida**를 마스터했습니다.

오늘, 그 복수전을 시작합니다.

---

## ⚔️ The Solution: Trust Manager Killing

SSL Pinning 로직은 보통 `CertificatePinner`나 `TrustManager`라는 클래스에 구현되어 있습니다.
이 녀석들이 하는 일은 "서버 인증서가 이상하면 에러를 뿜어라(Exception)"입니다.

우리의 전략은 단순합니다.
**"에러를 뿜는 입(함수)을 틀어막는다."**

검증 함수를 후킹해서, 아무 검사도 하지 않고 그냥 **조용히 리턴(return)**하게 만들면 됩니다. 그러면 앱은 "어? 검사가 통과됐네?"라고 착각하고 통신을 시작합니다.

---

## 🛠️ Lab: Breaking the Fortress

### 1. Universal SSL Pinning Bypass Script
전 세계 해커들이 미리 짜놓은 훌륭한 스크립트들이 **Frida Codeshare**에 있습니다.
우리는 바퀴를 다시 발명할 필요 없이 이것을 가져다 쓰면 됩니다.

```bash
frida -U -l https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/ -f com.example.targetapp
```

이 스크립트는 안드로이드의 대표적인 통신 라이브러리들(OkHttp, TrustKit, Android Native 등)을 모두 뒤져서 검증 로직을 무력화시킵니다.

### 2. 과정
1.  **mitmproxy** 실행 (포트 8080).
2.  단말기 프록시 설정.
3.  **Frida**로 우회 스크립트 주입하며 앱 실행.
4.  앱 화면: "연결 성공!"
5.  mitmproxy 화면: **트래픽이 보이기 시작함!** 🎉

이 순간의 쾌감은 이루 말할 수 없습니다.

---

## 📝 10주차 과제: Objection으로 뚫기

지난주에 쓴 **Objection**에도 이 기능이 있습니다.

```bash
android sslpinning disable
```
이 명령어를 사용하여 실제 Pinning이 걸린 앱(테스트용 앱)의 트래픽을 mitmproxy에 띄워보세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** SSL Pinning을 우회하기 위해 후킹해야 하는 안드로이드의 인증서 검증 관련 주요 컴포넌트(클래스) 이름은 무엇인가요? (T...M...)
2.  **Q2.** Frida Codeshare에 있는 스크립트를 다운로드 없이 바로 로드하기 위해 사용하는 커맨드라인 옵션은? (`--codeshare` 또는 URL 사용법)
3.  **Q3.** Pinning 우회 스크립트의 핵심 로직은 검증 함수가 예외(Exception)를 던지지 않고 어떻게 종료되도록 하는 것인가요? (정상 리턴)

---

다음 주는 **자동화(Automation)**입니다.
mitmproxy와 Frida를 따로 쓰는 게 아니라, **연동**해서 씁니다.
Frida가 앱 내부에서 암호키를 빼내서 mitmproxy에게 던져주면,
mitmproxy가 암호화된 패킷을 그 키로 풀어서 보여주는 **꿈의 분석 환경**을 설계해 봅니다.

**Unpin the world.**
