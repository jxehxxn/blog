---
layout: post
title: "Offensive AD & APT Mastery: Week 1 - Orientation & Landscape"
---

환영합니다, 예비 레드팀 오퍼레이터 여러분.

저는 지난 30년간 글로벌 금융권과 Tech Giant의 보안을 책임지며, 수많은 APT 공격을 방어하고, 또 모의해킹(Red Teaming)을 통해 취약점을 찾아내 왔습니다.

보안 업계에는 "Active Directory(AD)를 지배하는 자가 기업을 지배한다"는 말이 있습니다. 전 세계 기업의 90% 이상이 AD를 사용하며, 랜섬웨어나 APT 공격의 99%는 결국 AD 관리자 권한(Domain Admin) 탈취를 목표로 합니다.

이 강의는 단순히 툴을 돌리는 '스크립트 키디'를 양성하는 과정이 아닙니다. **프로토콜 레벨(LDAP, Kerberos, RPC)**에서의 깊은 이해를 바탕으로, 실제 APT 그룹이 사용하는 전술(TTPs)을 구사하고, 이를 탐지/대응할 수 있는 **시니어 보안 전문가**를 목표로 합니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리의 여정은 기초 프로토콜 분석에서 시작하여, AD 침투, 권한 상승, 횡적 이동(Lateral Movement), 그리고 최종적으로 도메인 장악 및 은밀한 지속성 유지(Persistence)까지 이어집니다.

### Part 1: Foundations & LDAP (기초와 정찰)
*   **W1:** 오리엔테이션 & APT/AD 공격 트렌드 (MITRE ATT&CK)
*   **W2:** LDAP Fundamentals & Reconnaissance (Protocol, Queries, Tools)
*   **W3:** LDAP Injection & ACL Abuse (Advanced LDAP attacks)

### Part 2: Initial Access & Credential Theft (초기 침투와 계정 탈취)
*   **W4:** AD Architecture & Reconnaissance (PowerView, BloodHound)
*   **W5:** Credential Access I - No-auth Attacks (LLMNR/NBT-NS Poisoning, SMB Relay)
*   **W6:** Credential Access II - Kerberos Attacks (Kerberoasting, AS-REP Roasting)

### Part 3: Lateral Movement & Domain Dominance (장악과 확장)
*   **W7:** Lateral Movement Techniques (PsExec, WMI, DCOM, Pass-the-Hash/Ticket)
*   **W8:** Advanced Persistence (Golden/Silver Tickets, Skeleton Key)
*   **W9:** Domain Dominance (DCSync, DCShadow)

### Part 4: APT Simulation & Defense (방어와 시뮬레이션)
*   **W10:** APT Simulation (C2 Infrastructure, Evasion)
*   **W11:** Threat Hunting & Detection (Splunk, Sysmon, YARA)
*   **W12:** Capstone - Full Chain Attack & Defense Simulation

---

## 🎓 1주차 강의: The APT & AD Landscape

### 1. What is APT? (Advanced Persistent Threat)
단순한 바이러스 유포가 아닙니다. **"특정 목적을 가지고, 장기간 은밀하게, 지속적으로"** 공격하는 행위입니다.
*   **Advanced:** 제로데이(0-day) 취약점이나 고도화된 악성코드를 사용.
*   **Persistent:** 한 번 막혀도 포기하지 않고 다른 경로(백도어, 계정 탈취)로 재진입.
*   **Threat:** 단순 재미가 아닌, 국가 지원 해커나 범죄 조직 등 명확한 위협 주체.

### 2. Why Active Directory?
AD는 기업의 **"신원(Identity)과 자산(Asset)의 중앙 저장소"**입니다.
*   **SSO의 편리함 = 해커의 편리함:** 한 번 로그인하면 어디든 갈 수 있습니다.
*   **레거시의 늪:** 20년 전 프로토콜(NTLMv1, SMBv1)이 하위 호환성 때문에 아직도 살아있습니다.
*   **복잡성:** 너무 복잡해서 관리자조차 누가 어디에 권한이 있는지 모릅니다. (ACL Abuse의 온상)

### 3. The Kill Chain (MITRE ATT&CK)
우리는 이 프레임워크를 따라 움직일 것입니다.
1.  **Reconnaissance:** OSINT, LDAP Query.
2.  **Resource Development:** C2 서버 구축, 페이로드 제작.
3.  **Initial Access:** 피싱, VPN 취약점, SMB Relay.
4.  **Discovery:** AD 구조 파악 (BloodHound).
5.  **Credential Access:** Mimikatz, Kerberoasting.
6.  **Lateral Movement:** 옆 PC로 이동 (PsExec).
7.  **Privilege Escalation:** Domain Admin 획득.
8.  **Collection & Exfiltration:** 데이터 유출.

---

## 🛠️ Lab: Environment Setup (필수)

이 강의는 100% 실습 중심입니다. 다음 환경을 반드시 구축해 오세요. (가상머신 권장)

### 구성 요소 (Minimum Requirements)
1.  **Domain Controller (DC):** Windows Server 2016 or 2019. (RAM 2GB+)
2.  **Domain Member (Client):** Windows 10 Enterprise. (RAM 2GB+)
3.  **Attacker Machine:** Kali Linux. (RAM 2GB+)

### 네트워크 설정
*   모두 **Host-Only Network** (또는 NAT Network)로 구성하여 인터넷과 격리하되, 서로 통신 가능해야 합니다.
*   Windows 10은 DC를 DNS 서버로 설정하고 도메인에 조인(Join)해야 합니다.

*상세한 랩 구축 가이드는 별도 [보충 포스트]로 제공됩니다.*

---

## 📝 1주차 과제: MITRE ATT&CK Analysis

**목표:** 실제 APT 그룹의 전술을 분석하여 공격 흐름을 이해하세요.

1.  **MITRE ATT&CK** 사이트에서 `APT29` (Cozy Bear) 또는 `Lazarus Group`을 검색하세요.
2.  해당 그룹이 **'Credential Access'**와 **'Lateral Movement'** 단계에서 사용한 테크닉(Technique ID)을 3개씩 찾으세요.
3.  각 테크닉이 AD 환경에서 구체적으로 어떻게 동작하는지 1~2줄로 요약하여 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** APT 공격의 특징 중 'Persistent'가 의미하는 바는 무엇인가요? (침투 후 지속적인 접근 권한 유지 및 재침투)
2.  **Q2.** Active Directory 환경에서 공격자가 가장 노리는 최상위 권한 그룹의 이름은? (Domain Admins 또는 Enterprise Admins)
3.  **Q3.** 공격자의 행동(TTPs)을 단계별로 분류하고 매핑한 지식 베이스(Knowledge Base)의 이름은? (MITRE ATT&CK)

---

다음 주, 우리는 AD의 전화번호부이자 공격자들의 보물지도인 **LDAP**를 털러 갑니다.
단순한 조회가 아니라, 관리자도 모르는 정보를 캐내는 **LDAP Reconnaissance**의 정수를 보여드리겠습니다.

**Stay Stealthy.**
