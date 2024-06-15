---
layout: post
title: "[Splunk 12주차] 캡스톤 프로젝트: 실전 APT 대응 및 최종 평가 (심화 교재)"
date: 2024-05-17 09:00:00 +0900
categories: [Splunk, Security]
tags: [Splunk, Capstone, APT, Security Operations, Threat Hunting]
---

# 제 12장: 캡스톤 프로젝트 - 실전 APT 대응 및 SOC 운영 체계 구축

## 1. 서론: 왜 캡스톤인가?

지난 11주 동안 우리는 데이터 수집부터 SPL 쿼리 작성, 대시보드 시각화, 그리고 경보 설정까지 Splunk의 핵심 기능을 모두 섭렵했습니다. 하지만 파편화된 지식은 실전에서 무용지물입니다. 공격자는 여러분이 배운 순서대로 공격하지 않습니다. 그들은 가장 취약한 고리를 찾아 침투하고, 탐지를 회피하며, 목표를 달성할 때까지 숨어 지냅니다.

이번 캡스톤 프로젝트는 단순한 기술 테스트가 아닙니다. 가상의 거대 금융 기업 'K-Bank'를 배경으로 벌어지는 실제 침해 사고 시나리오를 바탕으로, 여러분이 직접 SOC(Security Operations Center) 분석가가 되어 위협을 탐지하고 대응하는 전 과정을 수행하게 됩니다. 이 과정은 약 3시간 분량의 집중적인 실습과 분석을 요구하며, 완료 시 여러분은 현업 수준의 보안 분석 역량을 갖추게 될 것입니다.

---

## 2. 시나리오 상세 설계: "오퍼레이션 미드나잇 섀도우 (Operation Midnight Shadow)"

### 2.1. 대상 기업 프로필: K-Bank (Korea Digital Bank)
- **업종**: 인터넷 전문 은행 (제1금융권)
- **인프라 규모**: 
    - 사용자 워크스테이션: 5,000대 이상 (Windows 10/11)
    - 서버 인프라: 1,200대 (Linux 70%, Windows Server 30%)
    - 데이터베이스: Oracle Exadata, MS-SQL Cluster (고객 정보 및 거래 내역)
- **보안 규정**: ISMS-P, PCI-DSS, 전자금융감독규정 준수 대상
- **네트워크 구성**:
    - **External Zone**: 대고객 웹 서비스 및 모바일 API 서버
    - **DMZ**: 메일 게이트웨이, VPN 서버, 외부 DNS
    - **Internal Zone**: 일반 임직원 업무망 (AD 기반 관리)
    - **Server Zone**: 내부 애플리케이션 서버 및 미들웨어
    - **DB Zone**: 가장 엄격하게 격리된 데이터베이스 구역 (MFA 필수)
    - **Management Zone**: 보안 장비 및 Splunk 인프라 (Indexer Cluster, Search Head)

### 2.2. 위협 행위자: ShadowGroup
- **특징**: 국가 배후의 지원을 받는 것으로 추정되는 지능형 지속 위협(APT) 그룹.
- **주요 TTPs (Tactics, Techniques, and Procedures)**:
    - 사회 공학적 기법을 활용한 정교한 피싱 메일.
    - Living-off-the-Land (LotL) 기법: PowerShell, WMI, Certutil 등 정상 도구 악용.
    - 탐지 회피를 위한 DNS 터널링 및 스테가노그래피 활용.
    - 권한 상승 후 LSASS 덤프를 통한 자격 증명 탈취.

### 2.3. 침해 사고 타임라인 (Minute-by-Minute)

| 시간 (T) | 단계 | 상세 행위 | 관련 로그 소스 |
| :--- | :--- | :--- | :--- |
| **09:00:00** | **Reconnaissance** | 공격자가 LinkedIn을 통해 K-Bank 인사팀 직원의 이메일 주소를 수집함. | N/A (OSINT) |
| **09:15:22** | **Initial Access** | '2024년 연봉 조정 안내.pdf'라는 제목의 피싱 메일이 발송됨. | Mail Gateway Log |
| **09:42:10** | **Execution** | 직원이 파일을 실행. 내장된 매크로가 PowerShell을 호출하여 C2 서버에서 페이로드를 다운로드함. | Sysmon (ID 1, 3) |
| **10:05:45** | **Persistence** | `schtasks.exe`를 사용하여 매일 새벽 2시에 실행되는 'Windows Update Assistant' 작업 등록. | Sysmon (ID 1), Security (4698) |
| **14:20:00** | **Credential Access** | 공격자가 `procdump.exe`를 사용하여 LSASS 프로세스 메모리를 덤프, 관리자 해시 탈취. | Sysmon (ID 10), Security (4663) |
| **15:30:15** | **Lateral Movement** | 탈취한 관리자 계정으로 RDP를 통해 DB Zone의 점프 호스트에 접속 시도. | Security (4624, LogonType 10) |
| **23:00:00** | **Collection** | 고객 정보 50GB를 압축하여 임시 폴더(`C:\Windows\Temp\backup.zip`)에 저장. | Sysmon (ID 11) |
| **02:00:00** | **Exfiltration** | DNS TXT 레코드를 활용한 터널링 기법으로 데이터를 외부로 유출 시작. | Stream:DNS, Firewall Log |

---

## 3. 데이터 파이프라인 구축 (Ingest & Normalization)

실전 대응의 첫 번째 단계는 정확한 데이터를 확보하는 것입니다.

### 3.1. Universal Forwarder (UF) 설정
모든 엔드포인트에 UF를 설치하고, `inputs.conf`를 통해 다음 로그를 수집해야 합니다.

```ini
# %SPLUNK_HOME%\etc\system\local\inputs.conf

[WinEventLog://Security]
disabled = 0
index = windows_logs
renderXml = 1

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = windows_logs
renderXml = 1

[monitor://C:\Logs\Network\*.log]
index = network_logs
sourcetype = firewall_csv
```

### 3.2. Sysmon 구성 (Advanced Configuration)
단순한 설치가 아니라, 공격자의 행위를 정밀하게 추적할 수 있는 XML 설정이 필요합니다. (SwiftOnSecurity 기반 권장)
- **Event ID 1**: Process Creation (CommandLine, ParentCommandLine 필드 필수)
- **Event ID 3**: Network Connection (DestinationIP, DestinationPort)
- **Event ID 7**: Image Loaded (DLL Injection 탐지용)
- **Event ID 10**: Process Access (LSASS 접근 감시)

### 3.3. Splunk Stream을 활용한 네트워크 가시성
DNS 터널링을 탐지하기 위해 패킷 레벨의 분석이 필요합니다. Splunk Stream App을 사용하여 DNS 쿼리(Query), 응답(Answer), 쿼리 유형(Query Type)을 추출합니다.

---

## 4. 탐지 로직 개발 (Advanced SPL)

각 공격 단계별로 오탐을 최소화하고 정탐을 극대화하는 쿼리를 작성합니다.

### 4.1. 초기 침투 및 실행 탐지 (LotL 기법)
공격자는 `powershell.exe`를 직접 실행하기보다 인코딩된 명령어나 `IEX`를 사용합니다.

```splunk
index=windows_logs sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
| eval cmd_len=len(CommandLine)
| where (CommandLine LIKE "%-enc%" OR CommandLine LIKE "%-EncodedCommand%" OR CommandLine LIKE "%IEX%" OR CommandLine LIKE "%DownloadString%")
| stats count min(_time) as first_seen max(_time) as last_seen by host, User, CommandLine, ParentCommandLine
| eval duration = last_seen - first_seen
| table host, User, CommandLine, ParentCommandLine, count
```

### 4.2. 지속성 유지 탐지 (Scheduled Tasks)
비정상적인 이름이나 경로에서 생성되는 예약 작업을 감시합니다.

```splunk
index=windows_logs EventCode=4698
| xmlkv Message
| eval TaskName = mvindex(split(TaskName, "\"), -1)
| search NOT (TaskName IN ("AdobeFlashPlayerUpdate", "GoogleUpdateTaskMachineCore", "MicrosoftEdgeUpdateTaskMachineCore"))
| table _time, host, TaskName, Command
```

### 4.3. 내부 이동 탐지 (RDP Brute Force & Lateral Movement)
단일 소스 IP에서 여러 목적지로 짧은 시간 내에 발생하는 RDP 접속을 추적합니다.

```splunk
index=windows_logs EventCode=4624 Logon_Type=10
| bucket _time span=15m
| stats dc(dest) as unique_targets values(dest) as target_list by _time, src_ip
| where unique_targets > 3
```

### 4.4. 데이터 유출 탐지 (DNS Tunneling)
DNS 쿼리의 길이와 엔트로피를 분석하여 터널링을 식별합니다.

```splunk
index=network_logs sourcetype="stream:dns"
| eval query_len = len(query)
| where query_len > 100
| stats count sum(query_len) as total_bytes by src_ip, query
| sort - total_bytes
| head 20
```

---

## 5. SOC 대시보드 및 경보 체계 (SOC Operations)

### 5.1. SOC 분석가용 통합 대시보드 설계
대시보드는 '상황 인식(Situational Awareness)'이 목적입니다.
- **Top Panel**: 현재 활성화된 고위험 경보 (Critical Alerts)
- **Middle Panel**: 공격 단계별 타임라인 (Kill Chain Visualization)
- **Bottom Panel**: 의심스러운 프로세스 트리 및 네트워크 트래픽 추이

### 5.2. 위험 기반 경보 (Risk-Based Alerting, RBA)
단일 이벤트로 경보를 울리면 '경보 피로(Alert Fatigue)'가 발생합니다.
- **전략**: 각 탐지 로직에 점수(Risk Score)를 부여합니다.
- **예**: PowerShell IEX(20점) + Scheduled Task(30점) + RDP Lateral(40점) = 합계 90점 초과 시 즉시 대응(Incident Response)팀 호출.

---

## 6. 채점 기준 (Detailed Grading Rubrics)

| 평가 영역 | 세부 항목 | 배점 | 평가 가이드라인 |
| :--- | :--- | :--- | :--- |
| **데이터 수집 (Ingest)** | 로그 소스 구성 및 CIM 준수 | 20점 | 모든 로그가 지정된 인덱스에 수집되고, 필드 추출이 CIM 표준에 맞게 정규화되었는가? |
| **탐지 로직 (Detection)** | 5단계 공격 탐지 SPL 작성 | 40점 | 각 단계별 공격을 정확히 탐지하는가? 오탐을 줄이기 위한 화이트리스트 처리가 되어 있는가? |
| **시각화 (Visualization)** | SOC 대시보드 가독성 | 20점 | 드릴다운(Drill-down) 기능이 포함되어 분석가가 상세 정보를 즉시 확인할 수 있는가? |
| **대응 및 보고 (Reporting)** | 사고 분석 보고서 및 경보 설정 | 20점 | 탐지된 위협의 근본 원인(Root Cause)을 파악하고 적절한 대응 방안을 제시했는가? |

---

## 7. 모범 답안 및 분석 가이드 (Model Answers)

### 7.1. PowerShell 탐지 심화 분석
단순히 `powershell.exe`를 찾는 것이 아니라, 부모 프로세스가 `winword.exe`나 `excel.exe`인 경우를 찾는 것이 핵심입니다. 이는 오피스 매크로를 통한 초기 침투를 의미하기 때문입니다.

### 7.2. DNS 터널링 탐지 팁
단순히 길이만 보는 것이 아니라, `query_type="TXT"` 또는 `query_type="NULL"`인 경우를 함께 필터링하면 정탐률이 비약적으로 상승합니다.

---

## 8. 결론: 데이터는 거짓말을 하지 않는다

12주간의 여정이 끝났습니다. 여러분이 작성한 SPL 한 줄, 대시보드의 차트 하나가 실제 기업의 자산을 지키는 방패가 됩니다. 보안은 기술의 영역이기도 하지만, 결국은 '끈기'와 '통찰력'의 영역입니다. 

데이터 속에 숨겨진 공격자의 흔적을 찾아내는 즐거움을 잊지 마십시오. 여러분은 이제 진정한 Splunk 보안 전문가입니다.

---
**강의 시간**: 180분 (이론 60분, 실습 90분, 평가 및 피드백 30분)
**준비물**: Splunk Enterprise 실습 환경, Sysmon 설치 패키지, 가상 공격 로그 데이터셋
