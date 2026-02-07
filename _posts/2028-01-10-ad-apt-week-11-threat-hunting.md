---
layout: post
title: "Offensive AD & APT Mastery: Week 11 - Threat Hunting & Detection (Splunk, Sysmon, YARA)"
---

"공격은 예술이고 방어는 과학이다."
지금까지 예술을 했으니 이제 과학을 할 차례입니다.

AD 환경에서 공격을 탐지하는 핵심은 **로그(Log)**입니다.
하지만 윈도우 기본 로그만으로는 부족합니다. **Sysmon**이 필요합니다.

---

## 1. Sysmon (System Monitor)

Sysinternals의 무료 도구지만, EDR급의 가시성을 제공합니다.
*   **Event ID 1:** 프로세스 생성 (명령줄 인자 포함! -> `powershell -enc ...` 탐지)
*   **Event ID 3:** 네트워크 연결 (C2 통신 탐지)
*   **Event ID 8:** CreateRemoteThread (프로세스 인젝션 탐지)

**설정(Config):** SwiftOnSecurity의 설정 파일을 베이스로 튜닝해서 씁니다.

---

## 2. SIEM (Splunk)

로그가 쌓이면 사람이 못 봅니다. 검색 엔진이 필요합니다.
우리는 Splunk(무료 버전)를 사용해 로그를 수집하고 분석합니다.

### SPL (Search Processing Language)
*   `index=windows EventCode=4624 LogonType=3` (네트워크 로그인)
*   `index=sysmon EventCode=1 Image="*powershell.exe" CommandLine="*DownloadString*"` (악성 PS 다운로드)

---

## 3. YARA

파일 기반 탐지의 표준입니다. 악성코드의 **패턴(String, Byte)**을 정의합니다.

```yara
rule Mimikatz_Detection {
    strings:
        $s1 = "sekurlsa::logonpasswords" wide ascii
        $s2 = "gentilkiwi" wide ascii
    condition:
        any of them
}
```

---

## 🛠️ Lab: Blue Team Operation

1.  **Sysmon 설치:** 윈도우 클라이언트에 Sysmon을 설치합니다.
2.  **공격 실행:** Mimikatz를 실행해서 `sekurlsa::logonpasswords`를 쳐봅니다.
3.  **로그 확인:** 이벤트 뷰어 -> `Applications and Services Logs` -> `Microsoft` -> `Windows` -> `Sysmon` -> `Operational`
    *   Mimikatz 실행 기록(Event ID 1)과 LSASS 접근 기록(Event ID 10)이 남았는지 확인합니다.

---

## 📝 11주차 과제: Hunting Rules

**목표:** 우리가 배웠던 공격 기법을 탐지하는 Splunk 쿼리나 Sysmon 규칙을 작성하세요.

1.  **Kerberoasting:** RC4 암호화(`0x17`)를 사용하는 TGS 요청 탐지. (Event ID 4769)
2.  **DCSync:** `DS-Replication-Get-Changes` 권한 접근 탐지. (Event ID 4662)
3.  **Lateral Movement:** `wmiprvse.exe`가 자식 프로세스로 `cmd.exe`나 `powershell.exe`를 실행하는 것 탐지. (Sysmon Event ID 1)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 윈도우 기본 로그보다 상세한 프로세스 생성, 네트워크 연결, 파일 생성 로그를 남기기 위해 사용하는 도구는? (Sysmon)
2.  **Q2.** 바이너리 파일 내의 특정 문자열이나 바이트 패턴을 기반으로 악성코드를 분류하고 탐지하는 룰 포맷은? (YARA)
3.  **Q3.** Splunk에서 데이터를 검색하고 분석하기 위해 사용하는 쿼리 언어의 약자는? (SPL - Search Processing Language)

---

다음 주, 대망의 마지막 주차입니다.
처음부터 끝까지, **Full Chain Attack**을 수행하고 방어하는 **Capstone Project**를 진행합니다.

**Hunt or be hunted.**
