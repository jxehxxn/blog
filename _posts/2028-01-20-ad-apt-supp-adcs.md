---
layout: post
title: "Offensive AD & APT Mastery: Supplement - AD CS Attack Vectors (Certified Pre-Owned)"
---

"Kerberos는 옛날 이야기입니다. 이제는 인증서(Certificate)의 시대입니다."

2021년, SpecterOps가 발표한 백서 **"Certified Pre-Owned"**는 보안 업계를 뒤집어 놓았습니다.
많은 기업들이 Kerberos는 열심히 방어했지만, **AD CS(Active Directory Certificate Services)**는 "그냥 인증서 발급해주는 서버 아냐?" 하고 방치해뒀기 때문입니다.

이 보충 강의에서는 AD CS의 설정 오류를 이용해 도메인 관리자 권한을 탈취하는 **ESC1**부터 **ESC8**까지의 기법을 다룹니다.

---

## 1. Why AD CS?

AD CS는 사용자나 컴퓨터에게 디지털 인증서를 발급해줍니다.
이 인증서는 **Kerberos TGT를 요청할 때 암호 대신 사용될 수 있습니다 (PKINIT).**

만약 해커가 **"나는 Administrator다"**라는 내용의 인증서를 발급받을 수 있다면?
비밀번호를 몰라도, NTLM 해시를 몰라도, **합법적인 관리자 TGT**를 얻을 수 있습니다.

---

## 2. Top Attack Vectors

### ESC1: Misconfigured Certificate Templates
가장 흔하고 위험한 취약점입니다.
*   **조건:**
    1.  도메인 유저가 인증서를 요청할 수 있음. (Enrollment Rights)
    2.  **`ENROLLEE_SUPPLIES_SUBJECT`** 플래그가 켜져 있음. (사용자가 "내 이름은 Administrator야"라고 SAN(Subject Alternative Name)을 직접 써낼 수 있음)
*   **공격:**
    1.  해커는 일반 유저로 로그인합니다.
    2.  취약한 템플릿을 이용해 **"SAN=Administrator"**인 인증서를 요청합니다.
    3.  CA는 의심 없이 서명해줍니다.
    4.  해커는 이 인증서로 PKINIT을 수행해 Administrator의 TGT를 얻습니다.

### ESC8: NTLM Relay to AD CS (Web Enrollment)
*   **조건:** AD CS 웹 등록 페이지(`http://<CA>/certsrv`)가 켜져 있고, NTLM 인증을 허용함.
*   **공격:**
    1.  Week 5에서 배운 **NTLM Relay**를 사용합니다.
    2.  DC(Domain Controller)의 인증 패킷을 가로채서 AD CS 웹 페이지로 릴레이합니다.
    3.  **"DC 컴퓨터 계정"** 명의의 인증서를 발급받습니다.
    4.  이 인증서로 DC 권한을 획득하고, DCSync를 수행합니다.

---

## 3. Tools: Certify & Certipy

### Windows (Certify)
C# 도구입니다.
```powershell
# 취약한 템플릿 스캔
.\Certify.exe find /vulnerable

# 인증서 요청 (ESC1)
.\Certify.exe request /ca:CA01.corp.local\corp-CA01-CA /template:VulnTemplate /altname:Administrator
```

### Linux (Certipy)
Impacket 스타일의 파이썬 도구입니다. Kali에서 주로 씁니다.
```bash
# 스캔
certipy find -u user@corp.local -p password -dc-ip 192.168.1.10

# 공격 (ESC1)
certipy req -u user@corp.local -p password -ca corp-CA01-CA -template VulnTemplate -upn Administrator@corp.local

# 인증서 -> TGT 변환
certipy auth -pfx administrator.pfx
```

---

## 4. Mitigation

1.  **Templates Audit:** 불필요한 `ENROLLEE_SUPPLIES_SUBJECT` 플래그 제거.
2.  **Disable NTLM on CA:** IIS에서 NTLM 인증을 끄거나, HTTPS 및 Extended Protection for Authentication (EPA) 활성화.
3.  **Monitor:** 인증서 발급 로그(Event ID 4886, 4887) 모니터링. 특히 Administrator 권한의 인증서 발급 주의.

---

이 기법을 모르면 최신 모의해킹에서 살아남을 수 없습니다.
**"인증서가 곧 티켓이다."** 명심하십시오.
