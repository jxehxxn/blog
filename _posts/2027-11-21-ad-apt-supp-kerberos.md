---
layout: post
title: "Offensive AD & APT Mastery: Supplement - Kerberos Protocol Deep Dive"
---

Week 4로 넘어가기 전에, AD 인증의 심장인 **Kerberos**를 짚고 가야 합니다.
Kerberos를 모르면 **Golden Ticket, Silver Ticket, Kerberoasting** 공격을 이해할 수 없습니다.

"티켓(Ticket) 기반 인증"이라는 말을 많이 들어보셨을 겁니다. 놀이공원 자유이용권과 똑같습니다.

---

## 1. The Actors

1.  **Client:** 사용자 (나).
2.  **KDC (Key Distribution Center):** 도메인 컨트롤러(DC). 매표소.
    *   **AS (Authentication Service):** 신분 확인소.
    *   **TGS (Ticket Granting Service):** 놀이기구 티켓 발급소.
3.  **Service:** 파일 서버, DB 서버. 놀이기구.

---

## 2. The 3-Step Process (The Dance)

### Step 1: AS-REQ & AS-REP (TGT 발급)
*   **나:** "저 Alice인데요, 로그인하고 싶어요." (비밀번호로 암호화된 시간값 전송)
*   **KDC(AS):** "Alice 맞네. 여기 **TGT (Ticket Granting Ticket)** 받아."
    *   **TGT:** "이 사람은 인증된 Alice임"이라고 적힌 자유이용권. **krbtgt 계정의 비밀번호 해시로 암호화됨.**
    *   나는 `krbtgt` 비번을 모르므로 TGT 내용을 볼 수도, 조작할 수도 없음.

### Step 2: TGS-REQ & TGS-REP (Service Ticket 발급)
*   **나:** "저 파일 서버(CIFS) 쓰고 싶어요. 여기 제 TGT 보여드릴게요."
*   **KDC(TGS):** (TGT를 까서 확인 후) "ㅇㅇ 인증된 사람이네. 여기 파일 서버용 **ST (Service Ticket)** 받아."
    *   **ST:** "Alice가 파일 서버 써도 됨"이라고 적힌 티켓. **파일 서버 서비스 계정의 비밀번호 해시로 암호화됨.**

### Step 3: AP-REQ (서비스 이용)
*   **나:** (파일 서버에게) "나 티켓(ST) 있어. 문 열어."
*   **파일 서버:** (자기 비번으로 ST를 까서 확인) "KDC가 발급한 거 맞네. 어서 옵쇼."

---

## 3. Vulnerability Points (공격 지점)

### AS-REP Roasting
*   **Step 1**에서, 일부 계정("Pre-Auth Not Required")은 암호화된 시간값을 안 보내도 KDC가 일단 암호화된 TGT 패킷을 던져줍니다. 해커는 이걸 받아서 오프라인에서 브루트 포스(Brute Force)하여 패스워드를 알아냅니다.

### Kerberoasting
*   **Step 2**에서, 도메인 유저라면 누구나 "파일 서버용 ST 주세요"라고 요청할 수 있습니다.
*   KDC는 의심 없이 ST를 줍니다.
*   이 ST는 **서비스 계정의 비번 해시**로 암호화되어 있습니다.
*   해커는 메모리에서 이 ST를 추출해서 오프라인 크랙하여 서비스 계정 비번을 알아냅니다.

### Golden Ticket
*   만약 해커가 **krbtgt 계정의 비밀번호 해시**를 알아냈다면?
*   KDC를 거치지 않고 **스스로 TGT를 만들 수 있습니다.**
*   "나는 Administrator다"라고 적힌 TGT를 위조하면(Golden Ticket), 10년이고 20년이고 도메인을 지배할 수 있습니다.

### Silver Ticket
*   특정 서비스 계정(예: SQL Service)의 해시를 안다면?
*   **스스로 ST를 만들 수 있습니다.**
*   KDC 인증 없이 바로 DB 서버에 접근 가능합니다.

---

## 4. Summary

Kerberos는 강력하지만, **"비밀번호 해시가 키(Key)로 사용된다"**는 점이 약점입니다.
해시가 털리면 티켓 위조가 가능해집니다.

이 원리를 이해해야 Week 6과 Week 8이 쉬워집니다.
