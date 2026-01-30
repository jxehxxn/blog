---
layout: post
title: "Network & App Hacking Mastery: Week 12 - Capstone Project & The Road Ahead"
---

축하합니다! 👏
12주간의 대장정을 마치고 여기까지 오신 여러분은 이제 더 이상 초보자가 아닙니다.
여러분은 네트워크를 지배하고(mitmproxy), 앱의 영혼을 조종하는(Frida) 기술을 가졌습니다.

이제 마지막 시험, **Capstone Project**를 통해 여러분의 실력을 증명할 시간입니다.

---

## 🏆 The Capstone Project

여러분의 타겟은 **OWASP UnCrackable App** (또는 이와 유사한 취약점 점검용 앱)입니다.
이 앱에는 여러 단계의 방어막과 숨겨진 비밀(Secret String)이 있습니다.

### Mission Objectives (임무)

1.  **Level 1: Root Detection Bypass**
    *   앱을 켜면 "Rooted Device Detected"라며 꺼집니다.
    *   Frida/Objection을 사용해 이 체크를 우회하고 메인 화면에 진입하세요.
2.  **Level 2: Debugger Detection Bypass**
    *   앱이 분석 도구가 붙어있는지 감지합니다. 이것도 우회하세요.
3.  **Level 3: Secret String Extraction**
    *   앱 어딘가에 숨겨진 "비밀 문자열"을 입력해야 통과되는 구간이 있습니다.
    *   소스 코드를 정적 분석하거나, `strcmp` 같은 비교 함수를 후킹하여 정답을 알아내세요.
4.  **Bonus: Traffic Decryption**
    *   앱이 서버와 통신할 때 SSL Pinning을 뚫고 패킷을 캡처하여 내용을 확인하세요.

### Deliverables (제출물)
*   **Write-up:** 어떤 기술을 써서 어떻게 뚫었는지 과정을 상세히 기록한 리포트.
*   **Script:** 사용한 Frida 스크립트 (`solve.js`) 및 mitmproxy 스크립트.

---

## ⚖️ Ethics & Law (중요!)

힘에는 책임이 따릅니다.
우리가 배운 기술은 **양날의 검**입니다.

1.  **허가받지 않은 앱을 해킹하지 마세요.** (게임, 금융 앱 등) -> **법적 처벌 대상입니다.**
2.  **Bug Bounty 프로그램**을 이용하세요. (HackerOne, Bugcrowd 등) -> 기업이 허용한 범위 내에서 해킹하고, 취약점을 제보하면 **상금(Bounty)**을 받습니다. 이것이 화이트해커의 길입니다.
3.  **학습 목적으로만 사용하세요.**

---

## 🛣️ The Road Ahead

이 강의는 끝났지만, 해킹 공부는 끝이 없습니다.
다음 단계로 무엇을 공부하면 좋을까요?

*   **Native Deep Dive:** ARM Assembly를 배워서 C/C++ 리버싱 실력을 키우세요. (IDA Pro, Ghidra)
*   **OS Internals:** 안드로이드와 iOS 운영체제 커널 구조를 공부하세요.
*   **Game Hacking:** 유니티(Unity), 언리얼(Unreal) 엔진의 구조를 파헤쳐 보세요.

---

## 👋 Closing

처음 `mitmproxy`를 설치하고 신기해하던 1주차의 모습이 기억나시나요?
이제 여러분은 실행 중인 메모리를 조작하고 암호화를 무력화하는 수준에 도달했습니다.

기술은 계속 변합니다. 어제의 뚫는 법이 내일은 막힐 수 있습니다.
하지만 **"어떻게 작동하지?"**라고 끊임없이 질문하고 파고드는 **해커의 태도(Mindset)**만 있다면, 어떤 방어벽도 여러분을 막을 수 없을 것입니다.

여러분의 앞날에 무한한 `Hook Success`가 함께 하기를!

**Hack the planet (Legally).**

---
*Professor. Antigravity*
