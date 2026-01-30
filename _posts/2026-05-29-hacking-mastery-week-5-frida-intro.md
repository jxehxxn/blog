---
layout: post
title: "Network & App Hacking Mastery: Week 5 - Frida, The Scalpel for Runtime"
---

Part 2의 시작입니다.
네트워크에서 막혔다면(Pinning), 앱 내부로 들어가야 합니다.
앱이 실행되는 **그 순간(Runtime)**에 코드를 바꿔치기하는 마법, **Frida**를 소개합니다.

---

## 💉 What is Frida?

**Frida**는 **DBI (Dynamic Binary Instrumentation)** 툴킷입니다.
어려운 말 같죠? 쉽게 말해 **"실행 중인 프로세스에 자바스크립트 엔진을 주사기로 꽂아넣는 도구"**입니다.

주사기가 꽂히면 우리는 자바스크립트를 이용해 앱의 모든 것을 제어할 수 있습니다.
*   "이 함수 호출될 때 인자값 보여줘."
*   "이 함수 실행되지 말고 그냥 리턴해."
*   "숨겨진 변수값 읽어와."

이 모든 게 앱을 끄지 않고 **실시간**으로 이루어집니다.

---

## 🏗️ Architecture

Frida는 **Client-Server** 모델로 작동합니다.

1.  **Frida Server:** 모바일 기기(안드로이드/iOS) 안에서 돌아갑니다. 앱의 프로세스에 기생하며 명령을 기다립니다.
2.  **Frida Client:** 여러분의 PC(파이썬/CLI)입니다. 서버에게 "이 자바스크립트 코드 실행해줘"라고 명령을 보냅니다.

> **참고:** 루팅되지 않은 기기에서는 **Frida Gadget**이라는 `.so` 파일을 앱에 심어서 실행하기도 합니다.

---

## 🛠️ Lab: Hello Frida

아직 모바일 연결이 어렵다면, PC에서 간단히 연습해 봅시다.

### 1. 설치
```bash
pip install frida-tools
```

### 2. 타겟 프로그램 준비
간단한 계산기나, 파이썬 스크립트를 하나 띄워둡니다. 프로세스 ID(PID)를 알아두세요.

### 3. 후킹 스크립트 작성 (`hook.js`)
```javascript
// 모든 함수 호출을 추적하는 건 너무 많으니, 개념적으로만 봅니다.
console.log("Hello Frida! I am inside the process.");
```

### 4. 주입 (Injection)
```bash
frida -p [PID] -l hook.js
```
프로그램이 실행 중인데, 터미널에 "Hello Frida!"가 찍히는 것을 확인하세요. 외부에서 내부로 침투한 것입니다.

---

## 📝 5주차 과제: 환경 구축

본격적인 모바일 해킹을 위해 다음 환경을 구축해 오세요.

1.  **에뮬레이터 설치:** LDPlayer, Nox, 또는 안드로이드 스튜디오의 AVD를 설치하세요. (루팅 모드가 되는 것을 추천합니다.)
2.  **Frida Server 설치:** 에뮬레이터의 아키텍처(x86 또는 arm64)에 맞는 `frida-server` 바이너리를 다운받아 `/data/local/tmp/`에 넣고 실행 권한(`chmod +x`)을 주세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 실행 중인 프로그램의 바이너리를 수정하지 않고, 동작을 분석하거나 조작하는 기술을 약어로 무엇이라 하나요? (D...)
2.  **Q2.** 모바일 기기 내에서 실행되며, PC의 명령을 받아 앱을 조작하는 데몬 프로세스의 이름은?
3.  **Q3.** Frida가 앱 프로세스 내부에 주입하는 스크립트 언어는 무엇인가요?

---

다음 주는 안드로이드의 심장, **Java Layer**를 해부합니다.
"비밀번호가 틀렸습니다"라는 메시지를 띄우는 함수를 찾아서, 무조건 "성공"으로 바꾸는 실습을 진행합니다.

**Inject your logic.**
