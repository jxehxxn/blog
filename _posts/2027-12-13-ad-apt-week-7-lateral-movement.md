---
layout: post
title: "Offensive AD & APT Mastery: Week 7 - Lateral Movement Techniques (PsExec/WMI/PTH)"
---

계정 하나를 털었다고 끝이 아닙니다. 그 계정은 보통 말단 직원의 것입니다.
우리의 목표는 **Domain Admin**입니다.

해커는 말단 PC에서 시작해서, 서버 A로, 서버 B로, 그리고 마침내 도메인 컨트롤러(DC)로 점프합니다.
이 과정을 **Lateral Movement (횡적 이동)**라고 합니다.

오늘은 윈도우가 제공하는 관리 도구를 악용하여(Living off the Land), 흔적을 최소화하며 이동하는 법을 배웁니다.

---

## 1. PsExec: The Classic

Sysinternals의 PsExec는 원래 관리 도구지만, 해커에게는 최고의 무기입니다.
*   **원리:** 타겟 PC의 `ADMIN$` 공유 폴더에 서비스를 설치하고 실행합니다.
*   **단점:** 백신(AV)에 너무 잘 걸립니다.
*   **Impacket psexec.py:** 바이너리를 올리지 않고(Fileless) 메모리상에서 실행하는 기법을 씁니다.

```bash
psexec.py corp.local/admin:password@192.168.1.10
```

---

## 2. WMI (Windows Management Instrumentation)

PsExec보다 은밀합니다. 윈도우 관리 인프라를 이용합니다.
*   **장점:** 서비스 설치 과정이 없어서 로그가 적게 남습니다.
*   **도구:** `wmiexec.py` (Impacket)

```bash
wmiexec.py corp.local/admin:password@192.168.1.10
```
이러면 반화형 쉘(Semi-interactive shell)을 얻을 수 있습니다.

---

## 3. Pass-the-Hash (PtH)

비밀번호를 몰라도 됩니다. **NTLM 해시**만 있으면 됩니다.
Mimikatz나 다른 도구로 메모리에서 해시를 추출했다면, 그걸로 바로 이동합니다.

```bash
# 비밀번호 대신 해시 사용
wmiexec.py -hashes :<NTLM_HASH> corp.local/admin@192.168.1.10
```

---

## 4. WinRM (Windows Remote Management)

PowerShell Remoting에 사용되는 프로토콜입니다. (포트 5985/5986)
최근 윈도우 서버는 기본적으로 켜져 있습니다.
*   **도구:** `evil-winrm`
*   **특징:** PowerShell을 쓰기 때문에 AMSI(Antimalware Scan Interface) 우회 기법이 필요할 수 있습니다.

---

## 🛠️ Lab: Jumping Around

1.  **시나리오:** 여러분은 PC-A를 장악했고, 로컬 관리자 권한을 얻었습니다.
2.  **Dump:** Mimikatz로 메모리를 뒤져서 다른 유저(Admin)의 NTLM 해시를 찾습니다. (`sekurlsa::logonpasswords`)
3.  **Move:** 찾은 해시를 이용해 `wmiexec.py`로 PC-B(서버)에 접속합니다.
4.  **Confirm:** `hostname`과 `whoami`를 쳐서 내가 이동했음을 확인합니다.

---

## 📝 7주차 과제: Lateral Movement Matrix

**목표:** 상황별 최적의 이동 도구를 선택하는 능력을 기르세요.

다음 상황에서 PsExec, WMI, WinRM, RDP 중 무엇을 쓰는 게 가장 좋을지(탐지 확률 낮음, 성공 확률 높음) 선택하고 이유를 서술하세요.

1.  **상황 A:** 타겟 서버의 445 포트(SMB)만 열려있고, 백신이 강력하게 돌고 있다.
2.  **상황 B:** 타겟 서버의 5985 포트가 열려있고, GUI 접근은 불가능하다.
3.  **상황 C:** GUI로 접속해서 특정 프로그램을 실행하고 화면을 봐야 한다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** PsExec가 타겟 시스템에서 명령을 실행하기 위해 접근하는 관리 공유 폴더의 이름은? (ADMIN$)
2.  **Q2.** 비밀번호 대신 NTLM 해시값만을 이용하여 인증을 수행하는 공격 기법은? (Pass-the-Hash / PtH)
3.  **Q3.** WMI를 이용한 횡적 이동 시 주로 사용하는 포트는? (TCP 135, 445, 그리고 Dynamic Ports)

---

다음 주, 드디어 DC를 장악했다고 가정합니다.
이제 쫓겨나지 않기 위해 **Golden Ticket**을 만들고, 죽어도 되살아나는 **Persistence**를 심습니다.

**Move silently.**
