---
layout: post
title: "[Splunk 12주차] 캡스톤 프로젝트: 실전 APT 대응 및 최종 평가"
date: 2024-05-17 09:00:00 +0900
categories: [Splunk, Security]
tags: [Splunk, Capstone, APT, Security Operations]
---

벌써 12주가 지났습니다. 그동안 배운 모든 기술을 쏟아부을 시간입니다. 이번 주는 단순한 실습이 아닙니다. 실제 현장에서 벌어지는 지능형 지속 위협(APT) 시나리오를 바탕으로 여러분의 실력을 증명해야 합니다.

빅테크 보안 현장에서 30년 동안 수많은 침해 사고를 겪으며 느낀 점은 하나입니다. 도구는 거들 뿐, 결국 데이터를 읽는 눈이 승패를 가릅니다. 오늘 여러분은 그 눈을 증명하게 될 것입니다.

### 시나리오: "오퍼레이션 미드나잇 섀도우 (Operation Midnight Shadow)"

가상의 금융 기업 'K-Bank'가 공격 대상입니다. 공격 그룹은 국가 배후의 지원을 받는 것으로 추정되는 'ShadowGroup'입니다. 이들은 다음과 같은 단계로 침투를 시도했습니다.

1. **초기 침투 (Initial Access)**: 인사팀 직원에게 '2024년 연봉 조정 안내'라는 제목의 피싱 메일을 보냈습니다. 첨부된 PDF 파일에는 악성 매크로가 포함되어 있습니다.
2. **실행 (Execution)**: 직원이 파일을 여는 순간, 백그라운드에서 PowerShell 스크립트가 실행되어 외부 C2 서버에서 2단계 페이로드를 다운로드합니다.
3. **지속성 유지 (Persistence)**: 공격자는 시스템 재부팅 후에도 권한을 유지하기 위해 'Windows Update Assistant'라는 이름의 가짜 예약 작업(Scheduled Task)을 등록했습니다.
4. **내부 이동 (Lateral Movement)**: 탈취한 관리자 계정 정보를 이용해 RDP(원격 데스크톱)로 고객 정보가 담긴 데이터베이스 서버에 접속했습니다.
5. **데이터 유출 (Exfiltration)**: 대량의 고객 정보를 탐지 회피를 위해 DNS 터널링(DNS Tunneling) 기법을 사용하여 외부로 빼돌렸습니다.

### 프로젝트 요구사항

여러분은 K-Bank의 보안 운영 센터(SOC) 분석가로서 다음 과제를 완수해야 합니다.

1. **데이터 파이프라인 구축**: Universal Forwarder를 통해 Windows 이벤트 로그, Sysmon 로그, 네트워크 트래픽 로그를 Indexer로 수집하세요.
2. **탐지 로직 개발 (SPL)**: 위 시나리오의 5단계 공격을 각각 탐지할 수 있는 SPL 쿼리를 작성하세요.
3. **보안 대시보드 제작**: 실시간으로 공격 상황을 한눈에 파악할 수 있는 SOC 대시보드를 만드세요.
4. **경보 설정 (Alerting)**: 데이터 유출과 같은 치명적인 상황 발생 시 즉시 담당자에게 알림이 가도록 설정하세요.

---

### 채점 기준 (Grading Rubrics)

| 항목 | 평가 기준 | 배점 |
| :--- | :--- | :--- |
| **데이터 온보딩 및 정규화** | CIM(Common Information Model)을 준수하여 데이터를 필드 추출하고 정규화했는가? | 30점 |
| **탐지 정확도 및 효율성** | 작성한 SPL이 오탐(False Positive)을 최소화하면서 공격을 정확히 잡아내는가? | 40점 |
| **대시보드 가독성 및 활용성** | 보안 관제 요원이 즉각적인 의사결정을 내릴 수 있도록 시각화가 직관적인가? | 30점 |

---

### 모범 답안 (Model Answers)

#### 1. PowerShell 악성 스크립트 다운로드 탐지 (SPL)
```splunk
index=windows_logs sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
| search CommandLine="*powershell*" AND (CommandLine="*IEX*" OR CommandLine="*DownloadString*")
| table _time, host, User, CommandLine, ParentCommandLine
```
*설명: Sysmon 이벤트 ID 1번(프로세스 생성)을 활용해 PowerShell의 위험한 함수 사용을 추적합니다.*

#### 2. DNS 터널링 의심 징후 탐지 (SPL)
```splunk
index=network_logs sourcetype="stream:dns"
| eval query_length=len(query)
| where query_length > 100
| stats count by src_ip, query
| sort - count
```
*설명: 비정상적으로 긴 DNS 쿼리 길이를 계산하여 터널링 시도를 식별합니다.*

#### 3. 비정상적인 RDP 내부 이동 경보 설정
- **조건**: 동일한 소스 IP에서 10분 이내에 3개 이상의 서로 다른 목적지 서버로 RDP 접속 시도 성공.
- **설정**: `index=windows_logs EventCode=4624 Logon_Type=10 | stats dc(dest) as dest_count by src_ip | where dest_count >= 3`

---

### 최종 자가 진단 퀴즈 (Self-Assessment Quiz)

**Q1. APT 공격의 '지속성 유지' 단계에서 주로 사용되는 윈도우 메커니즘은 무엇인가요?**
- 정답: 예약 작업(Scheduled Tasks), 레지스트리 런 키(Registry Run Keys), 서비스 등록 등.

**Q2. Splunk에서 대량의 데이터를 처리할 때 검색 속도를 높이기 위해 사용하는 명령어는?**
- 정답: `tstats` 또는 `tscollect`를 활용한 가속화.

**Q3. DNS 터널링을 탐지할 때 가장 유의 깊게 살펴봐야 할 필드는?**
- 정답: 쿼리 문자열의 길이(Query Length)와 쿼리 유형(Query Type, 예: TXT 레코드).

**Q4. CIM을 사용하는 주된 이유는 무엇인가요?**
- 정답: 서로 다른 소스의 데이터를 공통된 필드명으로 통일하여 분석의 일관성을 유지하기 위함.

**Q5. SOC 대시보드에서 가장 중요한 요소는 무엇인가요?**
- 정답: 실시간성(Real-time)과 행동 지침(Actionable Insights) 제공.

---

12주 동안 고생 많으셨습니다. 이 프로젝트를 마친 여러분은 이제 단순한 툴 사용자가 아니라, 데이터를 통해 위협을 사냥하는 진정한 보안 전문가입니다. 현장에서 뵙겠습니다.
