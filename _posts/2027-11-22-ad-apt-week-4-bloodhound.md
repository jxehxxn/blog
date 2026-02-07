---
layout: post
title: "Offensive AD & APT Mastery: Week 4 - AD Architecture & Reconnaissance (BloodHound)"
---

지난주까지는 LDAP 프로토콜 레벨에서 놀았습니다.
이번 주부터는 진짜 **"Active Directory를 해킹"**합니다.

AD 해킹의 시작과 끝은 **BloodHound**입니다.
이 툴은 AD의 복잡한 권한 관계를 그래프로 시각화하여 **"내가 APC를 털면 B유저의 권한을 얻고, 그걸로 DC까지 갈 수 있다"**는 공격 경로(Attack Path)를 네비게이션처럼 보여줍니다.

---

## 1. AD Architecture Deep Dive

### Domain vs Forest
*   **Domain:** 보안 경계의 기본 단위. (`corp.local`)
*   **Forest:** 하나 이상의 도메인 집합. 신뢰 관계(Trust)로 연결됨. (`corp.local` <-> `subsidiary.local`)
*   **Trust:** "내 친구의 친구는 내 친구다." 트러스트가 연결된 다른 도메인으로 넘어가는 것이 공격의 핵심입니다.

### OUs and GPOs
*   **OU (Organizational Unit):** 사용자/컴퓨터를 담는 그릇. 여기에 **GPO (Group Policy Object)**를 겁니다.
*   **GPO Abuse:** "모든 개발팀 PC의 로컬 관리자 그룹에 'DevAdmin'을 추가하라"는 정책이 있다면? 해커는 'DevAdmin'만 먹으면 모든 개발팀 PC를 장악합니다.

---

## 2. BloodHound: The Attack Map

### Ingestor (SharpHound)
데이터 수집기입니다.
```powershell
.\SharpHound.exe -c All --zipFileName loot.zip
```
이 한 줄이면 도메인의 모든 정보를 긁어옵니다. (단, DC와 통신해야 하므로 로그가 남을 수 있습니다. 조심!)

### Analysis (Neo4j)
수집된 `loot.zip`을 BloodHound GUI에 업로드합니다.

**Key Queries:**
*   **Find Shortest Paths to Domain Admins:** 가장 빠른 승리 경로.
*   **Find Principals with DCSync Rights:** 황금 열쇠를 가진 자들.
*   **Find AS-REP Roastable Users:** 맛있는 먹잇감들.

---

## 3. PowerView Recon (Manual)

BloodHound는 시끄럽습니다. 조용히 하고 싶다면 PowerView를 씁니다.

```powershell
# 로컬 관리자 그룹 찾기 (Lateral Movement 대상)
Find-LocalAdminAccess

# 현재 세션이 있는 컴퓨터 찾기 (User Hunting)
Invoke-UserHunter
```

---

## 🛠️ Lab: Mapping the Attack Path

1.  **SharpHound 실행:** Windows Client에서 SharpHound를 돌려 데이터를 수집합니다.
2.  **BloodHound 실행:** Kali에서 neo4j와 bloodhound를 실행하고 데이터를 넣습니다.
3.  **경로 분석:**
    *   `Domain Admins` 노드를 우클릭 -> **Shortest Path to Here** 선택.
    *   어떤 경로가 나오나요?
    *   예: `Alice` -> `HasSession` -> `Workstation 1` -> `MemberOf` -> `Local Admins` -> ...

---

## 📝 4주차 과제: Analyzing the Map

**목표:** BloodHound 예제 데이터(또는 직접 구축한 랩 데이터)를 분석하여 공격 시나리오를 작성하세요.

1.  **Scenario:** 당신은 현재 `Guest` 권한밖에 없는 `Contractor` 계정을 탈취했습니다.
2.  **Path:** BloodHound에서 `Contractor` 노드로부터 `Domain Admins`까지 가는 경로를 찾으세요.
3.  **Explain:** 경로에 포함된 엣지(Edge)들의 의미를 설명하세요.
    *   예: `ForceChangePassword`, `AdminTo`, `HasSession`, `AddMember` 등.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** BloodHound가 데이터를 수집하기 위해 사용하는 수집기(Ingestor)의 이름은? (SharpHound / AzureHound)
2.  **Q2.** BloodHound 그래프에서 "사용자 A가 컴퓨터 B에 로그인해 있음"을 나타내는 엣지(Edge)는? (HasSession)
3.  **Q3.** 여러 도메인 간의 인증을 가능하게 해주는 AD의 연결 고리를 무엇이라 하나요? (Trust Relationship)

---

지도를 얻었으니 이제 문을 따러 갑니다.
다음 주, **인증 없이(No-Auth)** 자격 증명을 훔치는 마술 같은 기법, **LLMNR Poisoning**과 **SMB Relay**를 배웁니다.

**Follow the graph.**
