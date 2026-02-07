---
layout: post
title: "Offensive AD & APT Mastery: Week 8 - Advanced Persistence (Golden/Silver Tickets)"
---

도메인 관리자(Domain Admin) 권한을 얻었습니다. 축하합니다.
하지만 보안팀이 눈치채고 비밀번호를 바꾸면요? 다시 처음부터 해킹해야 할까요?

아니요. APT 그룹은 한 번 들어오면 절대 나가지 않습니다.
**Golden Ticket**과 **Silver Ticket**은 그 영구적인 권한의 상징입니다.

---

## 1. The Golden Ticket (TGT Forgery)

도메인 내의 모든 티켓을 발급해주는 주체는 **KDC**입니다.
KDC는 `krbtgt`라는 계정의 비밀번호 해시로 티켓을 서명합니다.

만약 우리가 `krbtgt`의 해시를 훔친다면?
**우리가 KDC가 되는 것입니다.**

*   유효기간 10년짜리 티켓 발급 가능.
*   존재하지 않는 유저 이름으로 티켓 발급 가능.
*   그룹 ID를 조작해서 "나는 Domain Admins다"라고 주장 가능.
*   비밀번호를 바꿔도 소용없음 (krbtgt 비번을 안 바꾸는 한).

**제작 (Mimikatz):**
```bash
kerberos::golden /user:FakeUser /domain:corp.local /sid:<DomainSID> /krbtgt:<Hash> /id:500
```

---

## 2. The Silver Ticket (ST Forgery)

Golden Ticket은 너무 강력해서 로그가 남을 수 있습니다. (TGT 발급 로그 등)
Silver Ticket은 좀 더 은밀합니다.

*   특정 서비스(예: SQL Server) 계정의 해시만 알면 됩니다.
*   KDC를 거치지 않고(TGT 없이), 바로 **서비스 티켓(ST)**을 위조해서 서버에 제출합니다.
*   **PAC Validation**이 켜져 있지 않으면 무적입니다.
*   DC에 로그가 전혀 안 남습니다!

---

## 3. Skeleton Key

이건 좀 올드하지만 무섭습니다.
DC의 `lsass.exe` 프로세스에 악성코드를 주입해서 **마스터키**를 심습니다.

*   원래 비밀번호: 동작함.
*   마스터키 (`mimikatz`): 동작함.

즉, 모든 유저의 비밀번호를 몰라도 마스터키 하나로 로그인할 수 있게 됩니다.

---

## 🛠️ Lab: Forging Tickets

1.  **DCSync:** (Week 9 미리보기) Mimikatz의 `lsadump::dcsync` 명령어로 `krbtgt` 계정의 NTLM 해시를 추출합니다.
2.  **Golden Ticket:** 위 해시를 이용해 TGT를 생성하고, 현재 세션에 주입(`ptt`)합니다.
3.  **Test:** `dir \\dc01\c$` 명령어가 먹히는지 확인합니다. (원래 권한이 없는 계정에서)

---

## 📝 8주차 과제: Detection Strategy

**목표:** Golden Ticket 공격을 탐지하는 방법을 조사하세요.

1.  Golden Ticket은 보통 유효기간을 10년으로 설정합니다. 이를 탐지할 수 있는 이벤트 로그 분석 방법은?
2.  `krbtgt` 계정의 비밀번호를 주기적으로 변경해야 하는 이유는? (한 번 유출되면 끝장나니까)
3.  Silver Ticket 공격 시 DC에 로그가 남지 않는다면, 어디서 로그를 봐야 탐지할 수 있을까요? (타겟 서비스가 돌아가는 서버의 로컬 이벤트 로그)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Golden Ticket을 생성하기 위해 반드시 알아야 하는 핵심 계정의 이름은? (krbtgt)
2.  **Q2.** Silver Ticket이 KDC(Domain Controller)와 통신하지 않고도 특정 서버에 접근할 수 있는 이유는? (서비스 계정의 비밀번호 해시만으로 서비스 티켓을 직접 서명/암호화하기 때문)
3.  **Q3.** 골든 티켓 공격을 무력화하기 위해 기업이 주기적으로(예: 180일마다) 수행해야 하는 절차는? (krbtgt 계정의 비밀번호를 두 번 연속 변경 - 이전 티켓 무효화)

---

다음 주, 도메인 컨트롤러를 속여서 모든 비밀번호를 복제해오는 기술, **DCSync**와 **DCShadow**를 배웁니다.
이것이 진정한 **Domain Dominance**입니다.

**Forever yours.**
