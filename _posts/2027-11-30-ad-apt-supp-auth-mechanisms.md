---
layout: post
title: "Offensive AD & APT Mastery: Supplement - Windows Authentication Mechanisms (NTLM vs Kerberos)"
---

Week 5에서 NTLM Relay를 배웠고, Week 6에서는 Kerberos 공격을 배울 겁니다.
이 둘은 윈도우 인증의 양대 산맥입니다. 헷갈리면 공격도 방어도 못 합니다.

확실하게 정리해드립니다.

---

## 1. NTLM (New Technology LAN Manager)

### 특징
*   **Challenge-Response 방식:**
    1.  클라이언트: "로그인 할래."
    2.  서버: "이 난수(Challenge)를 네 비번 해시로 암호화해서 보내봐."
    3.  클라이언트: (암호화해서 보냄 - Response)
    4.  서버: (DC한테 물어보거나 로컬 SAM과 비교해서) "맞네."
*   **구식:** 보안 취약점이 많습니다 (Relay, Pass-the-Hash).
*   **사용처:** IP로 접속할 때 (`\\192.168.1.10`), 워크그룹 환경.

### 취약점
*   **No Mutual Auth:** 서버가 진짜 서버인지 검증 안 함. (그래서 Relay 당함)
*   **Weak Algo:** DES/MD4 같은 고대 암호화 사용.

---

## 2. Kerberos

### 특징
*   **Ticket 기반:** (Supplement 1 참고)
*   **Mutual Auth:** 클라이언트도 서버를 믿고, 서버도 클라이언트를 믿음.
*   **사용처:** 도메인 환경, 호스트네임으로 접속할 때 (`\\server01.corp.local`).

### 취약점
*   **Time Sensitive:** 시간이 5분 이상 차이 나면 동작 안 함.
*   **Ticket Reuse:** 티켓(TGT)만 털리면 비번 몰라도 됨. (Pass-the-Ticket)

---

## 3. The "Downgrade" Attack

해커는 Kerberos보다 NTLM을 좋아합니다. 더 털기 쉬우니까요.
그래서 **강제로 NTLM을 쓰게 만듭니다.**

*   DNS 이름 대신 **IP 주소**를 쓰면 윈도우는 Kerberos를 포기하고 NTLM으로 전환합니다.
*   이 점을 이용해 `ntlmrelayx` 같은 도구가 활개 칠 수 있는 것입니다.

---

## 4. Pass-the-Hash (PtH) vs Pass-the-Ticket (PtT)

이름은 비슷하지만 완전히 다릅니다.

*   **PtH (NTLM):** 메모리에 있는 **NTLM Hash**를 훔쳐서, 비밀번호 입력 없이 바로 인증.
    *   도구: Mimikatz `sekurlsa::pth`
*   **PtT (Kerberos):** 메모리에 있는 **TGT/ST 티켓**을 훔쳐서(파일로 저장), 내 세션에 주입.
    *   도구: Mimikatz `kerberos::ptt`

**핵심:**
내가 털고 있는 PC에 "Domain Admin"이 로그인했던 흔적이 있다면?
*   그의 Hash를 줍거나 (PtH)
*   그의 Ticket을 주워서 (PtT)
*   DC로 날아갈 수 있습니다.

이것이 **Lateral Movement**의 기본 원리입니다.
