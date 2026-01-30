---
layout: post
title: "Network & App Hacking Mastery: Week 1 - Orientation & HTTP/HTTPS Interception"
---

환영합니다, 예비 보안 전문가 여러분.

저는 지난 30년간 글로벌 빅테크 기업에서 보안 아키텍처를 설계하고, 수많은 애플리케이션의 취약점을 분석해온 여러분의 가이드입니다. 오늘부터 12주간, 우리는 단순한 이론 공부가 아닌, **실제 네트워크 트래픽을 가로채고(mitmproxy), 실행 중인 앱의 뇌를 조작하는(Frida)** 실전 해킹의 세계로 떠납니다.

"앱이 어떻게 서버랑 통신하지?"
"게임 아이템 가격을 내 맘대로 바꿀 수 있을까?"
"SSL Pinning 때문에 패킷이 안 보이는데 어떡하지?"

이 모든 질문에 대한 답을 12주 후에 여러분은 스스로 내릴 수 있게 될 것입니다.
우리의 목표는 **Zero to Hero**입니다. 기초부터 시작해 현업에서 바로 쓸 수 있는 고급 기술까지 마스터해 봅시다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리의 여정은 크게 세 파트로 나뉩니다.

### Part 1: Network Traffic Analysis (mitmproxy)
네트워크 위에서 일어나는 일을 낱낱이 파헤칩니다.
*   **1주차:** 오리엔테이션 & HTTP/HTTPS 인터셉션 기초 (The "Man-in-the-Middle")
*   **2주차:** mitmproxy 심화 (파이썬 스크립팅, 커스텀 애드온 작성)
*   **3주차:** 고급 트래픽 조작 (Tampering, Replaying, 공격 자동화)
*   **4주차:** 방어 기법 우회 이론 (Certificate Pinning, Client-side checks)

### Part 2: Runtime Instrumentation (Frida) - Basics
네트워크가 막혔다면, 앱의 내부로 들어갑니다.
*   **5주차:** Frida 입문 & 아키텍처 (Client-Server, GumJS)
*   **6주차:** Android 후킹 기초 (Java 레이어, 메소드 추적 및 조작)
*   **7주차:** iOS 후킹 기초 (Objective-C/Swift 레이어 이해)
*   **8주차:** Native 후킹 (C/C++, libc 함수, 메모리 조작)

### Part 3: Advanced Techniques & Real-world Scenarios
두 도구를 결합하여 철옹성 같은 앱을 무력화합니다.
*   **9주차:** 루팅/탈옥 탐지 우회 (Prison Break)
*   **10주차:** SSL Pinning 우회 (mitmproxy와 Frida의 연합 작전)
*   **11주차:** 모의해킹 자동화 (Custom Tooling)
*   **12주차:** 캡스톤 프로젝트 (Target 앱 분석 및 PoC 작성)

---

## 🎓 1주차 강의: The Man in the Middle (중간자 공격)

### 1. HTTP vs HTTPS, 그리고 TLS Handshake
여러분이 브라우저 주소창에 `https://google.com`을 입력하면 무슨 일이 일어날까요?
내 컴퓨터와 구글 서버 사이에 **암호화된 터널**이 뚫립니다. 이 터널 안을 지나가는 데이터는 아무도 엿볼 수 없습니다. 해커가 중간에서 데이터를 가로채도, 알아볼 수 없는 난수표만 보일 뿐이죠.

하지만 우리가 모의해킹을 하려면 이 내용을 봐야 합니다. 어떻게 해야 할까요?

### 2. 프록시(Proxy)의 원리
우리는 **프록시(Proxy)**라는 대리인을 세울 겁니다.
*   **기존:** 나 <---> 구글
*   **프록시:** 나 <---> **[mitmproxy]** <---> 구글

mitmproxy는 나에게는 "내가 구글이야"라고 거짓말하고, 구글에게는 "내가 사용자야"라고 거짓말을 합니다. 양쪽 모두를 속이는 것이죠. 이것이 바로 **MITM(Man-in-the-Middle)**입니다.

### 3. CA 인증서 (The Fake ID)
그런데 HTTPS는 똑똑합니다. "너 구글 아니잖아! 인증서 내놔!"라고 요구합니다.
그래서 우리는 **가짜 신분증(Custom CA Certificate)**을 만들어야 합니다.
그리고 내 컴퓨터(또는 폰)에게 이렇게 명령합니다.
**"이 가짜 신분증을 발급한 기관(mitmproxy)을 무조건 신뢰해!"**

> **비유:**
> *   **HTTP:** 우체부가 엽서를 배달합니다. 누구나 내용을 읽을 수 있습니다.
> *   **HTTPS:** 007 가방에 넣어서 자물쇠를 잠궈 배달합니다. 열쇠는 수신자만 가지고 있습니다.
> *   **MITM:** 내가 가짜 자물쇠를 채워서 보냅니다. 중간에서 내가 열어보고(내 열쇠로), 다시 진짜 자물쇠로 잠궈서(서버 열쇠로) 보냅니다.

---

## 🛠️ 실습: mitmproxy 설치 및 설정

자, 이제 직접 해봅시다.

### Step 1: 설치
파이썬(Python 3.8+)이 설치되어 있어야 합니다. 터미널을 열고 입력하세요.

```bash
pip install mitmproxy
```

### Step 2: 실행 (Web UI)
초보자에게는 Web UI가 편합니다.

```bash
mitmweb
```
명령어를 치면 브라우저가 열리면서 `http://127.0.0.1:8081`에 mitmproxy 대시보드가 뜹니다.

### Step 3: 브라우저 프록시 설정
여러분의 브라우저가 mitmproxy(기본포트 8080)를 거치도록 설정해야 합니다.
크롬 확장 프로그램인 **SwitchyOmega**를 추천합니다.
*   Protocol: HTTP
*   Server: 127.0.0.1
*   Port: 8080

### Step 4: CA 인증서 설치 (가장 중요!)
프록시가 켜진 상태에서 브라우저 주소창에 `mitm.it`을 입력하세요.
*   운영체제에 맞는 인증서(pem, cer 등)를 다운로드합니다.
*   **"신뢰할 수 있는 루트 인증 기관(Trusted Root Certification Authorities)"** 저장소에 설치해야 합니다. (이 과정을 건너뛰면 빨간 경고창만 보게 됩니다.)

---

## 📝 1주차 과제: 검색어 조작하기

1.  mitmproxy를 켜고, 네이버나 구글 같은 검색 사이트에 접속합니다.
2.  검색창에 `Hello Security`라고 입력하고 검색 버튼을 누릅니다.
3.  mitmproxy 화면에서 해당 요청(Request)을 찾습니다. (`Intercept` 기능을 써보세요)
4.  보내지는 데이터(Body 또는 Query Param)를 `Hello Hacking`으로 수정해서 서버로 보냅니다.
5.  브라우저 화면에 `Hello Hacking`에 대한 검색 결과가 뜨는지 확인하세요.

성공하셨나요? 축하합니다. 여러분은 방금 첫 번째 해킹을 성공했습니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 클라이언트와 서버 사이에서 양쪽을 속여 통신을 가로채는 기법을 약어로 무엇이라고 하나요?
2.  **Q2.** HTTPS 통신을 가로채기 위해 클라이언트 기기에 반드시 설치해야 하는 파일은 무엇인가요?
3.  **Q3.** mitmproxy의 기본 리스닝 포트 번호는 몇 번인가요?

---

다음 주에는 mitmproxy의 진정한 강력함, **파이썬 스크립팅(Python Scripting)**을 배웁니다.
매번 손으로 데이터를 수정할 순 없겠죠? 코드로 자동화하는 법을 알려드리겠습니다.

**Start Intercepting.**
