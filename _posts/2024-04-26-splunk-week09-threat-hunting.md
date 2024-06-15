---
layout: post
title: "[Splunk 12주 완성] 9주차: 위협 헌팅과 MITRE ATT&CK 프레임워크 활용"
date: 2024-04-26 09:00:00 +0900
categories: splunk security
---

보안 운영 센터(SOC)에서 경보(Alert)만 기다리는 시대는 끝났습니다.
공격자는 이미 우리 네트워크 안에 들어와 있을 가능성이 큽니다.
단순히 탐지 규칙이 울리기를 기다리는 수동적 대응에서 벗어나야 합니다.
이번 주에는 스플렁크를 사용해 능동적으로 위협을 찾아내는 위협 헌팅을 다룹니다.

### 위협 헌팅의 철학과 방법론

위협 헌팅은 "이미 침해당했다(Assume Breach)"는 전제에서 시작합니다.
알려진 위협(Known Threats)은 SIEM의 탐지 규칙이 처리합니다.
하지만 알려지지 않은 위협(Unknown Threats)은 헌터의 몫입니다.
헌팅은 가설을 세우고 데이터를 뒤져 숨겨진 흔적을 찾는 과정입니다.

헌팅의 3대 요소는 다음과 같습니다.
1. 가설(Hypothesis): "공격자가 피싱 메일을 통해 침투한 뒤 권한을 상승시켰을 것이다."
2. 데이터(Data): 가설을 검증하기 위한 로그(Sysmon, Windows Event, Network Traffic).
3. 도구(Tools): 대량의 데이터를 빠르게 분석할 수 있는 스플렁크와 SPL.

### MITRE ATT&CK 프레임워크 심층 분석

공격자의 전술(Tactics)과 기법(Techniques)을 이해하는 것이 헌팅의 시작입니다.
MITRE ATT&CK은 공격자의 행동 양식을 체계화한 지식 베이스입니다.
스플렁크에서 수집하는 로그가 어떤 공격 단계에 해당하는지 매핑해야 합니다.

예를 들어, 윈도우 이벤트 로그 4624(로그온 성공)는 횡적 이동(Lateral Movement) 단계를 감시하는 핵심 데이터입니다.
Sysmon Event ID 1(Process Creation)은 실행(Execution)과 지속성(Persistence)을 확인하는 데 필수적입니다.
이러한 매핑을 통해 우리는 보안 가시성의 공백(Gap)을 파악하고 보완할 수 있습니다.

---

### [실전 시나리오] 2시간의 추적: 랜섬웨어 공격의 전 과정

이 시나리오는 피싱 메일 수신부터 최종 파일 암호화까지의 과정을 다룹니다.
헌터의 시점에서 각 단계별 가설을 세우고 SPL을 통해 위협을 식별해 보겠습니다.

#### 1단계: 초기 침투 (Initial Access) - 09:00 ~ 09:15
**가설:** 공격자가 악성 매크로가 포함된 워드 문서를 통해 침투했을 것이다.
**헌팅 포인트:** 오피스 프로그램(Word, Excel)이 비정상적인 자식 프로세스를 생성하는지 확인합니다.

```spl
index=sysmon EventCode=1 
    ParentImage="*\\winword.exe" OR ParentImage="*\\excel.exe"
    Image="*\\cmd.exe" OR Image="*\\powershell.exe" OR Image="*\\wscript.exe"
| table _time, host, user, ParentImage, Image, CommandLine
```
이 쿼리는 워드 문서가 실행된 뒤 명령 프롬프트나 파워쉘이 실행된 이력을 보여줍니다. 일반적인 업무 환경에서는 발생하지 않는 패턴입니다.

#### 2단계: 실행 및 C2 통신 (Execution & C2) - 09:15 ~ 09:30
**가설:** 실행된 파워쉘이 외부 서버에서 추가 페이로드를 다운로드하고 연결을 유지할 것이다.
**헌팅 포인트:** 파워쉘의 네트워크 연결 이력과 인코딩된 명령어를 분석합니다.

```spl
index=sysmon EventCode=3 Image="*\\powershell.exe"
| stats count dc(dest_ip) as target_ips values(dest_port) as ports by host, user, Image
| where target_ips > 0
```
네트워크 연결이 확인되었다면, 실행된 명령어의 내용을 상세히 들여다봅니다.

```spl
index=wineventlog EventCode=4104 
| eval script_block=ScriptBlockText
| search script_block="*FromBase64String*" OR script_block="*IEX*" OR script_block="*DownloadString*"
| table _time, host, user, script_block
```
파워쉘 스크립트 블록 로그(4104)를 통해 인코딩된 명령어나 다운로드 코드를 찾아냅니다.

#### 3단계: 권한 상승 및 자격 증명 탈취 (Privilege Escalation & Credential Access) - 09:30 ~ 10:00
**가설:** 공격자가 로컬 관리자 권한을 얻기 위해 LSASS 메모리를 덤프했을 것이다.
**헌팅 포인트:** lsass.exe 프로세스에 대한 비정상적인 접근을 감시합니다.

```spl
index=sysmon EventCode=10 TargetImage="*\\lsass.exe"
| stats count by SourceImage, GrantedAccess, host
| where GrantedAccess="0x1FFFFF" OR GrantedAccess="0x1010"
```
0x1FFFFF는 모든 권한을 의미하며, 미미카츠(Mimikatz) 같은 도구가 자격 증명을 추출할 때 사용합니다.

#### 4단계: 내부 정찰 (Discovery) - 10:00 ~ 10:20
**가설:** 공격자가 도메인 관리자 계정과 중요 서버를 찾기 위해 네트워크 명령어를 실행했을 것이다.
**헌팅 포인트:** 단시간 내에 실행된 정찰용 명령어들을 그룹화합니다.

```spl
index=sysmon EventCode=1 
    Image IN ("*\\whoami.exe", "*\\net.exe", "*\\ipconfig.exe", "*\\quser.exe", "*\\nltest.exe")
| stats count values(CommandLine) as commands dc(Image) as unique_tools by host, user, _time span=5m
| where unique_tools >= 3
```
5분 이내에 3개 이상의 정찰 도구가 실행되었다면 공격자의 활동으로 간주합니다.

#### 5단계: 횡적 이동 (Lateral Movement) - 10:20 ~ 10:45
**가설:** 탈취한 계정을 사용해 파일 서버로 RDP 또는 SMB 연결을 시도했을 것이다.
**헌팅 포인트:** 평소와 다른 소스 IP에서의 로그인 성공 이력을 찾습니다.

```spl
index=wineventlog EventCode=4624 Logon_Type IN (3, 10)
| stats count dc(dest) as target_count values(dest) as target_list by src_ip, user
| where target_count > 1
| sort - target_count
```
특정 사용자가 여러 대의 서버에 원격으로 로그인한 기록은 횡적 이동의 강력한 증거입니다.

#### 6단계: 데이터 유출 (Exfiltration) - 10:45 ~ 11:00
**가설:** 중요 데이터를 외부 클라우드 저장소로 전송했을 것이다.
**헌팅 포인트:** 대용량 아웃바운드 트래픽과 rclone 같은 도구의 실행을 확인합니다.

```spl
index=network_logs 
| stats sum(bytes_out) as total_out by src_ip, dest_ip
| where total_out > 104857600  # 100MB 이상
| sort - total_out
```
동시에 엔드포인트에서 유출 도구가 실행되었는지 교차 검증합니다.

```spl
index=sysmon EventCode=1 CommandLine="*rclone*" OR CommandLine="*pscp*"
| table _time, host, user, CommandLine
```

#### 7단계: 영향 및 암호화 (Impact) - 11:00 ~ 11:15
**가설:** 랜섬웨어가 실행되어 대량의 파일 확장자를 변경할 것이다.
**헌팅 포인트:** 단시간 내에 발생하는 대량의 파일 생성 및 삭제 이벤트를 추적합니다.

```spl
index=sysmon EventCode=11
| bin _time span=1m
| stats count as file_change_count values(TargetFilename) as files by host, _time
| where file_change_count > 50
```
1분 이내에 50개 이상의 파일이 변경되었다면 암호화 공격이 진행 중일 가능성이 큽니다.

---

### 고급 헌팅 기법: 데이터 소스 간 피벗(Pivoting)

위협 헌팅의 핵심은 서로 다른 로그를 연결하는 데 있습니다.
방화벽 로그에서 발견된 이상 IP를 엔드포인트 로그와 연결해 어떤 프로세스가 통신했는지 찾아야 합니다.

```spl
index=network_logs dest_ip="공격자_IP"
| join host [
    search index=sysmon EventCode=3
    | table host, Image, CommandLine, dest_ip
]
| table _time, host, Image, CommandLine, dest_ip
```
이 쿼리는 네트워크 로그와 Sysmon 로그를 결합해, 특정 외부 IP와 통신한 실제 프로세스가 무엇인지 정확히 짚어냅니다.

### 결론 및 향후 과제

위협 헌팅은 한 번으로 끝나는 작업이 아닙니다.
발견된 위협 패턴은 즉시 탐지 규칙(Alert)으로 변환해 자동화해야 합니다.
또한, 헌팅 과정에서 발견된 가시성의 공백은 로그 수집 정책을 수정해 보완해야 합니다.

스플렁크는 단순한 로그 저장소가 아닙니다.
헌터의 상상력을 현실로 구현해 주는 강력한 캔버스입니다.
오늘 배운 시나리오를 바탕으로 여러분의 환경에 맞는 가설을 세우고 데이터를 탐색해 보시기 바랍니다.

### 자가 진단 퀴즈
1. 위협 헌팅의 시작점인 '가설'을 세울 때 가장 많이 참고하는 프레임워크는?
2. Sysmon Event ID 10번이 감시하는 대상과 그 이유는?
3. 파워쉘의 인코딩된 명령어를 확인하기 위해 필요한 윈도우 이벤트 로그 번호는?

**정답:**
1. MITRE ATT&CK 프레임워크.
2. 프로세스 접근(Process Access). LSASS 덤프를 통한 자격 증명 탈취를 탐지하기 위함입니다.
3. Event Code 4104 (Script Block Logging).
