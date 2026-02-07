---
layout: post
title: "Offensive AD & APT Mastery: Week 2 - LDAP Fundamentals & Reconnaissance"
---

"AD 해킹을 하려면 뭐부터 해야 하나요?"
많은 초보자가 `nmap`부터 돌립니다. 하지만 고수는 **LDAP**부터 봅니다.

LDAP(Lightweight Directory Access Protocol)은 AD의 데이터베이스를 조회하는 프로토콜입니다. AD는 기본적으로 **"인증된 사용자에게 거의 모든 읽기 권한"**을 줍니다. 즉, 도메인 유저 계정 하나만 탈취하면, 회사의 전체 조직도, 서버 목록, 관리자 계정 정보 등을 합법적으로(?) 긁어갈 수 있다는 뜻입니다.

오늘은 이 **LDAP**을 이용해 조용하고 치명적인 정찰(Reconnaissance)을 수행하는 법을 배웁니다.

---

## 1. LDAP Fundamentals

### The Structure (DIT)
LDAP은 트리 구조입니다.
*   **DC (Domain Component):** `dc=corp,dc=local` (최상위)
*   **OU (Organizational Unit):** `ou=HR`, `ou=Servers` (폴더)
*   **CN (Common Name):** `cn=Administrator`, `cn=Users` (객체)

### Distinguished Name (DN)
객체의 전체 주소입니다.
`CN=Alice,OU=HR,DC=corp,DC=local`

### ObjectClass & Attributes
*   **user:** `sAMAccountName`(로그인ID), `memberOf`(그룹), `servicePrincipalName`(SPN)
*   **computer:** `operatingSystem`(OS버전)
*   **group:** `member`(속한 유저들)

---

## 2. PowerView: The Swiss Army Knife

PowerShell 기반의 AD 정찰 도구입니다. Kali에는 없으니 Windows Client에서 실행해야 합니다.

### Basic Commands
```powershell
# PowerView 로드
Import-Module .\PowerView.ps1

# 현재 도메인 정보 확인
Get-NetDomain

# 도메인 컨트롤러 확인
Get-NetDomainController

# 모든 사용자 목록 (엄청난 정보가 쏟아짐)
Get-NetUser

# 특정 그룹 멤버 확인 (Domain Admins 찾기!)
Get-NetGroupMember "Domain Admins"
```

---

## 3. LDAP Search Filters (Advanced)

PowerView 내부에서도 결국 LDAP 쿼리를 날립니다. 직접 쿼리를 짤 줄 알아야 합니다.

*   `(&(objectClass=user)(sAMAccountName=admin*))` : ID가 admin으로 시작하는 유저.
*   `(&(objectCategory=computer)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` : 활성화된 컴퓨터만.
*   **`(servicePrincipalName=*)`** : Kerberoasting 공격 대상(서비스 계정) 찾기. (Week 6 예습!)

---

## 4. SharpHound & BloodHound

텍스트로 보면 눈 아픕니다. 그래프로 그려주는 도구가 **BloodHound**입니다.
**SharpHound**라는 수집기(Ingestor)를 돌리면 LDAP을 싹 긁어서 JSON으로 만듭니다. 이걸 BloodHound에 넣으면 **"Domain Admin으로 가는 최단 경로"**를 그려줍니다.

"Alice 유저가 A서버의 관리자고, A서버에는 Bob이 로그인해 있고, Bob은 Domain Admins 그룹이다."
-> Alice를 털면 도메인을 먹을 수 있겠군!

---

## 🛠️ Lab: LDAP Reconnaissance

### 1. PowerView 실습
1.  Windows 10 Client에 도메인 유저로 로그인.
2.  PowerView를 다운로드하고 Import.
3.  `Get-NetUser | select sAMAccountName, description` 명령어로 **Description** 필드를 전수 조사하세요. (관리자가 비번을 적어놓는 경우가 꽤 있습니다!)

### 2. Linux에서 ldapsearch 사용 (Kali)
```bash
ldapsearch -x -H ldap://<DC_IP> -D "user@corp.local" -w "password" -b "dc=corp,dc=local" "(objectClass=user)"
```
GUI 도구인 `Apache Directory Studio`를 써보는 것도 좋습니다.

---

## 📝 2주차 과제: Hunting for Targets

**목표:** LDAP 쿼리를 이용해 다음 타겟을 식별하세요.

1.  **"Password Not Required"** 설정이 된 계정 찾기. (`userAccountControl` 속성 비트 연산 활용)
2.  **"Pre-Authentication Not Required"** 설정이 된 계정 찾기. (AS-REP Roasting 대상)
3.  OS 버전이 **Windows 7**이거나 **Server 2008**인 (취약한) 컴퓨터 찾기.

각 항목에 대한 LDAP Filter 구문을 작성하여 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** AD 환경에서 사용자, 컴퓨터, 그룹 등의 객체 정보를 저장하고 조회하기 위해 사용하는 프로토콜은? (LDAP)
2.  **Q2.** PowerView 명령어 중 현재 도메인의 모든 사용자를 나열하는 명령어는? (Get-NetUser)
3.  **Q3.** BloodHound가 AD 내의 관계를 시각화하기 위해 수집한 데이터를 저장하는 그래프 데이터베이스의 이름은? (Neo4j)

---

다음 주, 우리는 이 LDAP 프로토콜 자체의 취약점을 파고듭니다.
웹에 SQL Injection이 있다면, AD에는 **LDAP Injection**이 있습니다. 그리고 잘못된 권한 설정(ACL)을 악용하는 방법도 알아봅니다.

**Map the territory.**
