---
layout: post
title:  "Splunk 중급편 Part 1: [Hands-on] 엔드포인트 통합 가시성 확보"
date:   2025-07-08 13:00:00 +0900
categories: jekyll update
---

## 시작하며: 공격의 시작점, 엔드포인트를 파헤치다

Splunk 초급편에서는 Splunk의 기본 기능과 데이터 분석의 기초를 다졌습니다. 이제 중급편에서는 실제 보안 운영의 최전선, 바로 **엔드포인트(Endpoint)**에서 벌어지는 일들을 Splunk를 통해 어떻게 손금 보듯 들여다볼 수 있는지에 대해 깊이 있게 다뤄보겠습니다.

최신 사이버 공격의 90% 이상은 사용자의 PC, 즉 엔드포인트에서 시작됩니다. 사용자가 악성 이메일 첨부파일을 열거나, 위조된 웹사이트에 접속하는 순간 공격은 이미 시작된 것입니다. 따라서 엔드포인트에서 어떤 프로세스가 실행되고, 어떤 네트워크 통신이 발생하며, 어떤 파일이 생성되는지를 상세히 파악하는 것은 현대 보안의 핵심 과제입니다.

이번 파트에서는 여러분의 Windows PC를 강력한 보안 센서로 탈바꿈시키는 과정을 안내합니다. 기본 로그를 넘어, 공격의 흔적을 낱낱이 기록하는 **고급 Windows 로그**와 **Sysmon**을 설치하고, **주요 애플리케이션 로그**까지 수집하여 통합적인 가시성을 확보하는 방법을 실습해 보겠습니다.

## Step 1: 기본 그 이상, Windows 고급 로그 활성화

Windows는 기본적으로 많은 이벤트를 기록하지만, 보안 분석에 필수적인 몇 가지 중요한 로그는 비활성화되어 있습니다. 공격자들이 가장 애용하는 PowerShell의 상세 로깅을 활성화하여 공격의 실마리를 놓치지 않도록 설정해야 합니다.

**PowerShell 스크립트 블록 로깅 활성화 (로컬 그룹 정책 편집기)**

1.  `Win + R` 키를 눌러 `gpedit.msc`를 실행합니다.
2.  다음 경로로 이동합니다:
    `Computer Configuration > Administrative Templates > Windows Components > Windows PowerShell`
3.  오른쪽 설정 창에서 다음 두 가지 항목을 찾아 **Enabled**로 변경합니다.
    *   **Turn on Module Logging**: PowerShell 모듈(Cmdlet)의 파이프라인 실행 정보를 기록합니다.
    *   **Turn on PowerShell Script Block Logging**: 실행되는 모든 PowerShell 스크립트의 내용을 기록합니다. 난독화된 스크립트라도 실행 시점에는 복호화되므로, 악성 행위를 파악하는 데 결정적인 단서가 됩니다.

## Step 2: 프로세스의 모든 것을 기록하는 감시자, Sysmon 설치

**Sysmon(System Monitor)**은 Microsoft가 제공하는 무료 엔드포인트 행위 추적 유틸리티입니다. 기본 이벤트 로그가 기록하지 않는 프로세스 생성(부모-자식 관계 포함), 네트워크 연결, 파일 생성 시간 변경 등 공격 분석에 필수적인 상세 정보를 기록해 줍니다.

**1. Sysmon 다운로드 및 설정 파일 준비**

*   [Microsoft Sysinternals 사이트](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)에서 Sysmon을 다운로드합니다.
*   Sysmon은 설정 없이 설치하면 너무 많은 로그를 기록하여 오히려 분석을 방해할 수 있습니다. 따라서 보안 커뮤니티에서 널리 사용되는 Best Practice 설정 파일을 사용하는 것을 강력히 권장합니다. 대표적으로 [SwiftOnSecurity의 Sysmon 설정 파일](https://github.com/SwiftOnSecurity/sysmon-config)이 있습니다. 이 GitHub 페이지에서 `sysmonconfig-export.xml` 파일을 다운로드하여 Sysmon을 다운로드한 폴더에 함께 저장합니다.

**2. Sysmon 설치**

관리자 권한으로 PowerShell 또는 명령 프롬프트를 열고, Sysmon을 다운로드한 경로로 이동하여 다음 명령어를 실행합니다.

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml
```

이제 Sysmon은 백그라운드 서비스로 실행되며, `sysmonconfig-export.xml` 설정에 따라 시스템의 모든 행위를 `Microsoft-Windows-Sysmon/Operational` 이벤트 로그에 기록하기 시작합니다.

## Step 3: Universal Forwarder로 모든 로그 통합 수집

이제 활성화한 고급 로그와 Sysmon 로그, 그리고 다양한 애플리케이션 로그를 Splunk Universal Forwarder를 통해 수집할 차례입니다. Forwarder가 설치된 PC의 `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf` 파일을 열고 아래 내용을 추가/수정합니다.

**`inputs.conf` 통합 설정 예시 (Best Practice)**

```ini
# 1. Windows 기본 보안/시스템/애플리케이션 이벤트 로그
[WinEventLog://Security]
disabled = 0
index = win_event_logs

[WinEventLog://System]
disabled = 0
index = win_event_logs

[WinEventLog://Application]
disabled = 0
index = win_event_logs

# 2. PowerShell 고급 로그
[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
index = win_event_logs
sourcetype = WinEventLog:PowerShell

# 3. Sysmon 상세 행위 로그
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = win_sysmon_logs
sourcetype = xmlwineventlog:sysmon

# 4. Windows Defender 탐지 로그
[WinEventLog://Microsoft-Windows-Windows Defender/Operational]
disabled = 0
index = win_security_solution_logs
sourcetype = WinEventLog:Defender

# 5. (예시) Chrome 브라우저 방문 기록 수집 (Scripted Input)
# 아래 스크립트를 C:\Program Files\SplunkUniversalForwarder\etc\apps\my_app\bin\get_chrome_history.ps1 로 저장
# [script://."C:\Program Files\SplunkUniversalForwarder\etc\apps\my_app\bin\get_chrome_history.ps1"]
# disabled = 0
# interval = 3600  # 1시간마다 실행
# index = win_app_logs
# sourcetype = chrome:history
```

*   **인덱스 분리:** 데이터의 종류에 따라 `win_event_logs`, `win_sysmon_logs`, `win_security_solution_logs` 와 같이 인덱스를 분리하여 저장하면, 검색 성능과 데이터 관리 효율성을 크게 높일 수 있습니다. (Splunk 웹에서 해당 인덱스들을 미리 생성해야 합니다.)
*   **Source Type 지정:** PowerShell, Sysmon, Defender 로그는 Splunk가 더 잘 해석할 수 있도록 전용 Source Type을 지정해 주는 것이 좋습니다.
*   **스크립트 입력(Scripted Input):** Chrome 방문 기록처럼 파일 기반 로그가 아닌 경우, PowerShell 스크립트를 주기적으로 실행하여 그 결과를 Splunk로 보낼 수 있습니다. 이는 매우 강력한 기능으로, 거의 모든 종류의 데이터를 수집할 수 있게 해줍니다.

## 다음 이야기

이번 파트에서는 엔드포인트의 가시성을 극대화하기 위해 다양한 종류의 로그를 수집하고 Splunk로 전송하는 방법을 배웠습니다. 이제 여러분의 Splunk에는 단순한 로그가 아닌, 공격의 전체 스토리를 재구성할 수 있는 풍부한 컨텍스트 정보가 쌓이기 시작했습니다.

하지만 이 데이터들은 아직 제각각의 언어로 이야기하고 있습니다. 다음 파트에서는 **"데이터 인텔리전스 강화: CIM(Common Information Model)을 활용한 데이터 정규화"**를 주제로, 이 다양한 데이터들을 Splunk의 공통 언어로 통일하여 강력한 통합 분석의 기반을 마련하는 방법을 알아보겠습니다.

---
