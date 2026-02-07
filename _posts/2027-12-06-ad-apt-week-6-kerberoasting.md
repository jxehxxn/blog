---
layout: post
title: "Offensive AD & APT Mastery: Week 6 - Credential Access II - Kerberos Attacks (Roasting)"
---

"비밀번호를 몰라도 티켓을 달라고 할 수 있다고요?"
네, Kerberos 프로토콜의 설계상 허점입니다.

AD 환경에서 가장 흔하고, 가장 강력하며, 가장 막기 힘든 공격.
**Kerberoasting**과 **AS-REP Roasting**입니다.

---

## 1. Kerberoasting

**원리:**
1.  도메인 유저는 누구나 서비스(SQL, IIS 등)를 사용하기 위해 **서비스 티켓(ST)**을 요청할 수 있습니다.
2.  KDC는 ST를 발급해줍니다.
3.  이 ST는 **서비스 계정의 비밀번호 해시**로 암호화되어 있습니다.
4.  해커는 이 ST를 받아서, 서비스를 이용하는 척하다가 **집에 가져갑니다.**
5.  집에서 GPU 빵빵한 컴퓨터로 해시를 깹니다 (Crack).

**타겟:**
*   `ServicePrincipalName (SPN)`이 설정된 사용자 계정. (주로 `svc_sql`, `svc_backup` 등)
*   이런 계정들은 보통 비밀번호를 잘 안 바꿉니다. 그리고 권한은 셉니다.

---

## 2. AS-REP Roasting

**원리:**
1.  Kerberos 인증 첫 단계(AS-REQ)에서, 사용자는 원래 비밀번호로 암호화된 시간값을 보내야 합니다(Pre-Auth).
2.  하지만 **"Pre-Authentication Not Required"** 옵션이 켜진 계정은, 그냥 ID만 보내도 KDC가 TGT(의 일부)를 보내줍니다.
3.  이 응답(AS-REP) 메시지는 **사용자의 비밀번호 해시**로 서명되어 있습니다.
4.  해커는 이걸 받아서 깹니다.

**타겟:**
*   실수로 `Pre-Auth Not Required` 설정이 켜진 계정.

---

## 3. Tools: Impacket & Rubeus

### Linux (Impacket)
```bash
# Kerberoasting
GetUserSPNs.py corp.local/user:password -request

# AS-REP Roasting
GetNPUsers.py corp.local/ -usersfile users.txt -format hashcat
```

### Windows (Rubeus)
C#으로 작성된 툴로, 탐지 우회에 유리합니다.
```powershell
# Kerberoasting
.\Rubeus.exe kerberoast /outfile:hashes.txt

# AS-REP Roasting
.\Rubeus.exe asreproast /format:hashcat /outfile:hashes.txt
```

---

## 🛠️ Lab: Roast and Crack

1.  **설정:** AD에서 `svc_sql` 계정을 만들고 SPN을 설정합니다. `setspn -A MSSQL/server01:1433 svc_sql`
2.  **공격:** Kali에서 `GetUserSPNs.py`를 실행하여 해시를 추출합니다.
3.  **크랙:** 추출한 해시를 `hashcat`에 넣고 돌립니다.
    *   `hashcat -m 13100 -a 0 hashes.txt wordlist.txt`
4.  **성공:** 비밀번호가 평문으로 튀어나옵니다. 이제 이 계정으로 로그인하면 됩니다.

---

## 📝 6주차 과제: Defensive Measures

**목표:** Roasting 공격을 탐지하고 방어하는 전략을 수립하세요.

1.  **Defense:** 서비스 계정의 비밀번호를 어떻게 설정해야 Kerberoasting에 안전할까요? (길이, 복잡도, 주기적 변경 - MSA)
2.  **Detection:** 누군가 TGS-REQ를 비정상적으로 많이 요청(Honeytoken 등)하거나, RC4 암호화를 사용하는 티켓을 요청할 때 어떤 이벤트 로그 ID가 남나요? (4769, 4768)
3.  **Honeytoken:** 공격자를 유인하기 위한 가짜 SPN 계정을 만드는 방법을 제안하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Kerberoasting 공격의 대상이 되는 계정은 반드시 어떤 속성을 가지고 있어야 하나요? (ServicePrincipalName - SPN)
2.  **Q2.** AS-REP Roasting 공격이 가능한 이유는 사용자 계정에 어떤 플래그가 설정되어 있기 때문인가요? (Do not require Kerberos preauthentication)
3.  **Q3.** Kerberoasting으로 추출한 티켓 해시를 크랙하기 위한 Hashcat의 모드 번호는? (13100 - Type 23)

---

이제 계정(Credential)을 얻었습니다.
다음 주, 이 계정을 가지고 옆 PC, 그리고 서버로 이동하는 **Lateral Movement** 기술을 배웁니다.

**Roast them all.**
