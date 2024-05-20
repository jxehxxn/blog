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

### 수동적 대응 vs 능동적 위협 헌팅
대부분의 보안 팀은 SIEM이 생성하는 경보에 의존합니다.
이것은 이미 알려진 패턴에 대한 사후 대응입니다.
반면 위협 헌팅은 가설을 세우고 데이터를 뒤져 숨겨진 흔적을 찾는 과정입니다.
빅테크 기업들은 이 과정을 자동화하고 MITRE ATT&CK 프레임워크와 연계해 체계화합니다.

### MITRE ATT&CK 매핑의 중요성
공격자의 전술(Tactics)과 기법(Techniques)을 이해하는 것이 헌팅의 시작입니다.
스플렁크에서 수집하는 로그가 어떤 공격 단계에 해당하는지 매핑해야 합니다.
예를 들어, 윈도우 이벤트 로그 4624(로그온 성공)는 횡적 이동(Lateral Movement) 단계를 감시하는 핵심 데이터입니다.

### 실전 시나리오: 횡적 이동(Lateral Movement) 추적하기
공격자가 한 시스템을 장악한 뒤 다른 서버로 권한을 확장하는 과정을 찾아보겠습니다.
우리는 단시간 내에 여러 대상 시스템으로 시도된 비정상적인 원격 로그인이라는 가설을 세웁니다.

#### 1단계: 원격 로그인 로그 필터링
먼저 네트워크를 통한 로그인 시도(Logon Type 3)를 추출합니다.
```spl
index=wineventlog EventCode=4624 Logon_Type=3
| stats count dc(dest) as target_count by src_ip, user
```

#### 2단계: 임계치 설정 및 시각화
일반적인 사용자는 하루에 접속하는 서버 대수가 제한적입니다.
5대 이상의 서로 다른 서버에 접속한 이력을 필터링합니다.
```spl
index=wineventlog EventCode=4624 Logon_Type=3
| stats count dc(dest) as target_count values(dest) as target_list by src_ip, user
| where target_count > 5
| sort - target_count
```

#### 3단계: 프로세스 실행 로그와 결합(Pivoting)
로그인 직후 어떤 명령어가 실행되었는지 확인해야 합니다.
이것이 데이터 소스 간의 피벗(Pivoting)입니다.
```spl
index=wineventlog EventCode=4624 Logon_Type=3
| stats count dc(dest) as target_count values(dest) as target_list by src_ip, user
| where target_count > 5
| join user [
    search index=wineventlog EventCode=4688
    | table _time, user, New_Process_Name, Process_Command_Line
]
| table _time, src_ip, user, target_list, New_Process_Name, Process_Command_Line
```
이 쿼리는 대량의 서버 접속 후 실행된 프로세스를 한눈에 보여줍니다.
powershell.exe나 psexec.exe가 실행되었다면 즉각적인 조사가 필요합니다.

### 데이터 소스 간 피벗의 핵심
위협 헌팅의 묘미는 서로 다른 로그를 연결하는 데 있습니다.
방화벽 로그에서 발견된 이상 IP를 엔드포인트 로그와 연결해 어떤 프로세스가 통신했는지 찾아야 합니다.
스플렁크의 join, append, stats 명령어가 이 연결 고리 역할을 합니다.

### 자가 진단 퀴즈
1. 위협 헌팅과 일반적인 경보(Alert)의 가장 큰 차이점은 무엇인가요?
2. MITRE ATT&CK 프레임워크에서 횡적 이동을 감시하기 위해 가장 중요한 윈도우 이벤트 코드는?
3. 스플렁크에서 두 개 이상의 서로 다른 인덱스 데이터를 연결할 때 사용하는 대표적인 명령어는?

**정답:**
1. 경보는 알려진 위협에 대한 수동적 대응이고, 헌팅은 가설 기반의 능동적 탐색입니다.
2. Event Code 4624 (특히 Logon Type 3 또는 10).
3. join, stats, lookup.
