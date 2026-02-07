---
layout: post
title: "Offensive AD & APT Mastery: Week 9 - Domain Dominance (DCSync, DCShadow)"
---

도메인 컨트롤러(DC)에 RDP로 접속해서 `lsass.exe`를 덤프 뜨는 건 하수입니다. 백신에 걸리고 로그도 남습니다.
고수는 **DC인 척**합니다.

AD의 핵심 기능인 **복제(Replication)** 프로토콜을 악용하는 **DCSync**와 **DCShadow**를 배웁니다.

---

## 1. DCSync

AD는 여러 대의 DC가 있을 때 서로 데이터를 동기화합니다.
"나 DC2인데, 유저 A 비밀번호 바꼈어? 나한테도 좀 줘."

DCSync는 Mimikatz가 이 요청을 흉내 내는 것입니다.
*   **권한:** `Replicating Directory Changes` 권한이 필요합니다. (Domain Admins는 기본적으로 있음)
*   **결과:** DC에 접속하지 않고도 **원격에서 모든 계정의 비밀번호 해시를 받아옵니다.**

```bash
# Impacket
secretsdump.py corp.local/admin:password@192.168.1.10
```

---

## 2. DCShadow

DCSync가 데이터를 **빼오는(Get)** 거라면, DCShadow는 데이터를 **밀어넣는(Push)** 겁니다.

1.  공격자의 PC를 **가짜 DC**로 등록합니다.
2.  가짜 DC에서 "Admin 계정의 설명(Description) 필드를 'Hacked'로 바꿨어"라고 변경 사항을 만듭니다.
3.  진짜 DC에게 "복제해 가라"고 강제합니다.
4.  진짜 DC는 의심 없이 데이터를 받아들이고 변경 사항을 적용합니다.
5.  가짜 DC 등록을 해제하고 사라집니다.

**무서운 점:**
DCShadow가 수행하는 변경은 **이벤트 로그에 거의 남지 않습니다.** 보안 장비를 우회하는 끝판왕 기술입니다.

---

## 3. Protecting Against Sync Attacks

DCSync를 막으려면?
*   `Replicating Directory Changes` 권한이 누구에게 있는지 주기적으로 감사해야 합니다. (AdminSDHolder 확인)
*   DC 간의 복제 트래픽을 모니터링해야 합니다.

---

## 🛠️ Lab: Total Compromise

1.  **DCSync:** `secretsdump.py`를 이용해 도메인 전체 해시를 덤프 뜹니다.
2.  **Analysis:** `krbtgt` 해시와 `Administrator` 해시를 확인합니다.
3.  **Audit:** 덤프 뜬 파일에서 평문 비밀번호나 취약한 비밀번호를 사용하는 사용자를 식별합니다.

---

## 📝 9주차 과제: DCShadow Scenario

**목표:** DCShadow 공격의 이론적 시나리오를 구성하세요.

1.  당신은 Domain Admin 권한을 얻었습니다.
2.  특정 사용자의 `PrimaryGroupID`를 변경하여 권한을 상승시키려고 합니다.
3.  DCShadow를 이용해 이를 수행하는 절차를 Mimikatz 명령어로 정리하세요.
    *   `lsadump::dcshadow /object:Alice /attribute:primarygroupid /value:512 /push`
4.  이 공격이 왜 기존의 `net user` 명령어나 AD 사용자 관리 도구보다 탐지하기 어려운지 설명하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** DCSync 공격이 악용하는 윈도우의 정상적인 기능(프로토콜)은 무엇인가요? (Directory Replication Service Remote Protocol - DRSR)
2.  **Q2.** DCSync 공격을 수행하기 위해 공격자 계정에 반드시 필요한 두 가지 권한은? (Replicating Directory Changes, Replicating Directory Changes All)
3.  **Q3.** 공격자가 자신의 워크스테이션을 일시적으로 Rogue DC로 등록하여 악성 객체를 AD에 주입하는 공격 기법은? (DCShadow)

---

이제 공격 기술은 모두 배웠습니다.
Part 4에서는 이 모든 것을 종합하여 **C2(Command & Control)** 서버를 구축하고, 실제 APT 그룹처럼 **시뮬레이션**을 해봅니다.

**Become the Shadow.**
