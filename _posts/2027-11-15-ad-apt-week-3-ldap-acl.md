---
layout: post
title: "Offensive AD & APT Mastery: Week 3 - LDAP Injection & ACL Abuse"
---

Week 2에서는 정상적인 LDAP 조회(Recon)를 배웠습니다.
이번 주는 **비정상적인** LDAP 공격을 다룹니다.

웹 해킹에서 SQL Injection이 치명적이듯, LDAP을 사용하는 웹 애플리케이션(로그인 페이지, 직원 검색 등)은 **LDAP Injection**에 취약할 수 있습니다. 또한, AD 내부의 잘못된 권한 설정(ACL)은 권한 상승의 지름길입니다.

---

## 1. LDAP Injection

웹 앱이 사용자 입력을 받아 LDAP 필터를 동적으로 생성할 때 발생합니다.

**취약한 코드 예시 (Java):**
```java
String filter = "(&(uid=" + user_input + ")(objectClass=inetOrgPerson))";
ctx.search(base, filter, controls);
```

**공격 벡터:**
*   입력값: `*` -> `(&(uid=*)(objectClass=inetOrgPerson))`
    *   모든 사용자 검색됨. (인증 우회 가능)
*   입력값: `*)(uid=*))(|(uid=*` -> `(&(uid=*)(uid=*))(|(uid=*)(objectClass=inetOrgPerson))`
    *   필터 로직을 조작하여 권한 없는 정보 노출.

**대응:** 입력값 검증 및 인코딩 (이스케이프 처리).

---

## 2. ACL Abuse (Access Control List)

AD 객체마다 **"누가 이 객체를 건드릴 수 있는지"** 적혀있는 목록(ACL)이 있습니다.
*   **GenericAll:** 모든 권한. (비밀번호 변경 가능!)
*   **WriteDacl:** 권한 자체를 수정할 수 있는 권한. (내 맘대로 권한 추가 가능)
*   **WriteOwner:** 소유권을 가져올 수 있는 권한.

**시나리오:**
1.  해커가 `HelpDesk` 계정을 탈취함.
2.  분석해보니 `HelpDesk` 그룹이 `Domain Admins` 그룹 객체에 대해 `WriteOwner` 권한이 있음. (잘못된 설정)
3.  해커는 `Domain Admins` 그룹의 주인을 `HelpDesk`로 바꿈.
4.  이제 주인 마음대로 `Domain Admins`에 자기 계정을 추가함.
5.  **Game Over.**

---

## 3. Tools for ACL Abuse

이 복잡한 ACL을 눈으로 찾기는 불가능합니다.

*   **BloodHound:** "HelpDesk -> GenericAll -> Domain Admins" 같은 경로를 시각적으로 보여줍니다.
*   **PowerView:**
    ```powershell
    # 특정 객체의 ACL 확인
    Get-ObjectAcl -Identity "Administrator"
    
    # ACL 수정하여 권한 부여 (공격)
    Add-ObjectAcl -TargetIdentity "TargetUser" -PrincipalIdentity "Hacker" -Rights ResetPassword
    ```

---

## 🛠️ Lab: Exploiting ACLs

(BloodHound가 필요합니다)

1.  **Recon:** BloodHound를 돌려 `Outbound Control Rights`를 분석합니다.
2.  **Target Selection:** 내가 장악한 유저(Start Node)에서 Domain Admin(End Node)까지 이어지는 ACL 경로(Attack Path)를 찾습니다.
3.  **Exploit:**
    *   만약 `ForceChangePassword` 권한이 있다면?
    *   `net user <TargetUser> <NewPassword> /domain` 명령어로 타겟의 비번을 강제로 바꿔버리고 로그인합니다.

---

## 📝 3주차 과제: Blind LDAP Injection

**목표:** LDAP Injection을 통해 블라인드 방식으로 정보를 빼내는 쿼리를 구상하세요.

상황: 로그인 페이지에 ID를 입력하면 "존재함/존재하지 않음"만 리턴합니다.
1.  관리자 ID가 `admin`이라고 가정할 때,
2.  `admin`의 `telephoneNumber` 속성 첫 번째 자리가 `0`인지 확인하는 주입 구문은?
    *   힌트: `(&(uid=admin)(telephoneNumber=0*))` 가 참이면 "존재함", 거짓이면 "존재하지 않음".
3.  이 원리를 이용해 전화번호 전체를 알아내는 스크립트(Python) 로직을 서술하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** LDAP 쿼리에서 `*`, `(`, `)` 등의 특수 문자를 필터링하지 않을 때 발생하는 취약점은? (LDAP Injection)
2.  **Q2.** AD 객체에 대해 "비밀번호 초기화"를 할 수 있는 ACL 권한의 이름은? (ResetPassword / ForceChangePassword)
3.  **Q3.** BloodHound에서 "사용자 A가 그룹 B의 멤버는 아니지만, 그룹 B의 속성을 수정할 수 있는 권한"을 나타내는 엣지(Edge)는? (WriteDacl, GenericWrite 등)

---

다음 주, 이제 기초는 끝났습니다.
본격적인 침투를 시작합니다. 도메인 컨트롤러의 구조를 파헤치고, **초기 침투(Initial Access)** 시나리오를 설계합니다.

**Inject and Abused.**
