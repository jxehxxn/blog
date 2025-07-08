---
layout: post
title:  "Splunk 중급편 Part 3: [Hands-on] 위협 탐지 시나리오 구현"
date:   2025-07-08 14:00:00 +0900
categories: jekyll update
---

## 시작하며: 점들을 연결하여 공격의 그림을 그리다

지금까지 우리는 엔드포인트의 상세한 행위 데이터를 수집하고, CIM을 통해 표준화했으며, 데이터 모델로 분석을 가속화할 준비를 마쳤습니다. 이제 이 모든 준비 과정을 바탕으로, 보안 분석의 꽃이라 할 수 있는 **위협 탐지(Threat Detection)**를 수행할 차례입니다.

숙련된 공격자들은 단 하나의 로그만으로 탐지될 만한 어리석은 행동을 하지 않습니다. 그들은 여러 단계에 걸쳐, 정상적인 행위처럼 보이는 작은 활동들을 조합하여 최종 목표를 달성합니다. 따라서 우리도 흩어진 점(개별 로그)들을 연결하여 하나의 선(공격 시나리오)으로 파악하는 **상관 분석(Correlation Analysis)** 기반의 탐지 전략이 필요합니다.

이번 파트에서는 실제 공격 사례에서 영감을 얻은 세 가지 구체적인 위협 탐지 시나리오를 Splunk SPL을 통해 직접 구현하고, 이를 자동화된 경보로 만드는 방법을 실습합니다.

## 시나리오 1: "초기 침투" - 브라우저 다운로드와 의심스러운 자식 프로세스

*   **공격 개요:** 사용자가 인터넷에서 신뢰할 수 없는 파일(예: `invoice.zip`)을 다운로드하고 압축을 풀었더니, 그 안의 스크립트 파일(`run.bat`)이 실행되어 악성 PowerShell 명령을 실행하는 전형적인 초기 침투 기법입니다.
*   **탐지 전략:** "Chrome 브라우저가 파일을 다운로드한 후, 짧은 시간(예: 5분) 내에, 해당 파일과 관련된 프로세스(`explorer.exe` -> `cmd.exe` -> `powershell.exe`)가 실행되는 경우"를 탐지합니다.

**[Hands-on] SPL로 구현하기**

```spl
// 1. Chrome 브라우저의 파일 다운로드 이벤트(Sysmon EventCode 15: FileCreate)를 찾는다.
`sysmon` EventCode=15 Image="C:\Program Files\Google\Chrome\Application\chrome.exe"
| rename TargetFilename as downloaded_file

// 2. 다운로드된 파일의 경로에서 파일명을 분리한다.
| eval downloaded_file_name = mvindex(split(downloaded_file, "\\"), -1)

// 3. 다운로드 이벤트 시간과 파일명을 기준으로, 5분 내에 발생한 모든 프로세스 생성 로그와 결합(join)한다.
| join _time, downloaded_file_name [
    `sysmon` EventCode=1
    | eval process_time = _time
    | eval process_name = mvindex(split(Image, "\\"), -1)
    | search process_name IN (powershell.exe, cmd.exe, wscript.exe, cscript.exe)
]
| where process_time >= _time AND process_time <= (_time + 300)

// 4. 결과 테이블을 정리하여 보여준다.
| table _time, user, downloaded_file, process_name, CommandLine
```

이 SPL은 두 개의 다른 이벤트(파일 생성, 프로세스 생성)를 `join` 명령어로 연결하여 하나의 맥락으로 분석하는 상관 분석의 핵심을 보여줍니다.

## 시나리오 2: "내부 정찰" - 비정상적인 시스템 정보 수집 명령어 실행

*   **공격 개요:** 시스템에 침투한 공격자는 주변에 어떤 서버들이 있고, 현재 로그인한 사용자의 권한은 무엇인지 파악하기 위해 다양한 시스템 정보 수집 명령어를 실행합니다. (예: `whoami`, `netstat -an`, `systeminfo`, `net user`)
*   **탐지 전략:** "웹 서버나 DB 서버 등 일반 사용자가 직접 접속하여 명령어를 실행할 일이 없는 서버에서, 또는 웹 애플리케이션 서비스 계정(예: `IIS_USER`)이 이러한 내부 정찰 명령어를 실행하는 경우"를 탐지합니다.

**[Hands-on] SPL로 구현하기**

```spl
`sysmon` EventCode=1 

// 1. 알려진 내부 정찰 명령어들을 리스트업한다.
| search (CommandLine="*whoami*" OR CommandLine="*netstat*" OR CommandLine="*systeminfo*" OR CommandLine="*net user*" OR CommandLine="*net group*")

// 2. 일반 사용자가 아닌, 시스템/서비스 계정이 실행한 경우를 필터링한다.
| search (user="NT AUTHORITY\\SYSTEM" OR user="*SERVICE*" OR user="IIS_USER")

// 3. 중요 서버 그룹(예: 웹서버, DB서버)에서 발생한 경우를 필터링한다.
| search (host IN (WEB_SERVER_*, DB_SERVER_*))

// 4. 탐지된 이벤트의 상세 정보를 테이블로 보여준다.
| table _time, host, user, process_name, CommandLine
```

이 SPL은 공격의 '행위' 자체보다는, 그 행위가 발생한 '맥락'(누가, 어디서)을 조건으로 추가하여 오탐을 줄이고 탐지의 정확도를 높이는 방법을 보여줍니다.

## 시나리오 3: "데이터 유출" - 압축 프로그램을 이용한 대량 파일 접근

*   **공격 개요:** 공격자는 내부의 중요 문서들을 외부로 빼내기 전에, 탐지를 피하고 전송 용량을 줄이기 위해 압축 프로그램(예: `7z.exe`, `WinRAR.exe`)을 사용하여 파일들을 하나의 아카이브로 묶습니다.
*   **탐지 전략:** "평소에 사용되지 않던 압축 관련 프로세스가, 단시간(예: 1분)에 수백 개 이상의 파일에 접근(읽기)하는 행위"를 탐지합니다.

**[Hands-on] SPL로 구현하기**

```spl
// 1. 파일 접근 이벤트(Sysmon EventCode 11: FileCreate - 실제로는 파일 읽기/쓰기)를 대상으로 한다.
`sysmon` EventCode=11

// 2. 알려진 압축 프로그램 프로세스를 필터링한다.
| search (Image="*\\7z.exe" OR Image="*\\WinRAR.exe" OR Image="*\\zip.exe")

// 3. 1분 단위로, 어떤 프로세스가(Image), 어떤 호스트에서(host), 몇 개의 파일에(TargetFilename) 접근했는지 카운트한다.
| bucket _time span=1m
| stats dc(TargetFilename) as accessed_file_count by _time, host, Image

// 4. 파일 접근 횟수가 100개를 초과하는 경우만 필터링한다.
| where accessed_file_count > 100

| sort -accessed_file_count
```

이 SPL은 `bucket`과 `stats` 명령어를 활용하여 특정 시간 동안의 행위를 집계하고, 그 통계치를 기반으로 임계점(Threshold)을 넘어가는 이상 징후를 탐지하는 기법을 보여줍니다.

## 경보(Alert)로 자동화하기

위에서 만든 모든 SPL은 **Save As > Alert** 기능을 통해 자동화된 경보로 설정할 수 있습니다. (초급편 Part 6 참고) 예를 들어, 시나리오 3의 경우 "매 5분마다 검색을 실행하여, 파일 접근 횟수가 100개를 넘는 경우가 1건이라도 발견되면, 즉시 보안팀에 이메일 알림을 보내라" 와 같이 설정하여 24시간 자동 모니터링 체계를 구축할 수 있습니다.

## 다음 이야기

이번 파트에서는 상관 분석과 통계 기반의 이상 징후 탐지 기법을 활용하여, 고도화된 위협을 탐지하는 실전 시나리오를 구현해 보았습니다. 이제 여러분의 Splunk는 잠들지 않는 파수꾼이 되어 시스템을 지켜볼 것입니다.

그렇다면, 이 파수꾼이 경보를 울렸을 때 우리는 무엇을 해야 할까요? 다음 파트에서는 **"보안 분석가의 대응: 경보 조사 및 침해 분석(Drill-down) 워크플로우"**를 주제로, 발생한 경보가 실제 위협인지 판단하고, 피해 범위를 추적하는 구체적인 조사 과정을 단계별로 따라가 보겠습니다.

---
