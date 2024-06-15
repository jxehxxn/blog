---
layout: post
title: "[Splunk 12주 완성] 5주차: SIEM의 본질과 빅테크 보안 모니터링 (심화 교안)"
date: 2024-03-29 09:00:00 +0900
categories: splunk security
---

# 제 5장: SIEM의 본질과 엔터프라이즈 보안 아키텍처

빅테크 보안 현장에서 30년을 굴렀다. 수많은 툴이 명멸하는 걸 봤지만 SIEM은 여전히 보안 운영의 심장이다. 오늘은 단순한 툴 사용법이 아니라 빅테크 엔지니어가 어떻게 위협을 정의하고 탐지하는지 그 본질을 다뤄보자. 이 글은 단순한 블로그 포스트를 넘어 3시간 분량의 집중 강의를 텍스트로 옮긴 교안이다.

---

## 1. SIEM의 역사와 진화: 왜 Splunk인가?

많은 이들이 SIEM(Security Information and Event Management)을 그저 로그를 모아두는 창고로 생각한다. 큰 오산이다. SIEM의 진짜 가치는 흩어진 점들을 연결해 선을 만드는 '맥락(Context)'에 있다.

### 1.1 로그 관리 도구 vs SIEM
로그 관리 도구(Log Management)와 SIEM은 목적부터 다르다.
*   **로그 관리 도구 (ELK, Graylog 등)**: 데이터의 저장, 검색, 규정 준수(Compliance)를 위한 보관에 집중한다. "어제 무슨 일이 있었나?"라는 질문에 답한다.
*   **SIEM (Splunk, ArcSight, QRadar 등)**: 실시간 상관관계 분석(Correlation), 경보(Alerting), 사고 대응 워크플로우에 집중한다. "지금 무슨 일이 벌어지고 있으며, 얼마나 위험한가?"라는 질문에 답한다.

### 1.2 SIEM의 삼국지: ArcSight, QRadar, 그리고 Splunk
SIEM의 역사를 이해하면 왜 지금 Splunk가 대세인지 알 수 있다.

1.  **ArcSight (개척자 시대)**: 2000년대 초반 SIEM의 표준이었다. 강력한 룰 엔진을 가졌지만, Oracle 기반의 무거운 데이터베이스 구조와 'Schema-on-Write' 방식이 발목을 잡았다. 데이터를 넣기 전에 미리 필드를 정의해야 했기에, 새로운 로그 소스를 추가하는 데만 몇 주가 걸렸다.
2.  **QRadar (통합의 시대)**: IBM이 인수한 QRadar는 로그뿐만 아니라 네트워크 플로우(NetFlow) 데이터를 통합하며 한 시대를 풍미했다. 하지만 여전히 정해진 틀(Schema) 안에서만 움직여야 하는 경직성이 문제였다.
3.  **Splunk (파괴적 혁신)**: Splunk는 'Schema-on-Read'라는 개념을 들고 나왔다. 데이터를 일단 다 집어넣고, 검색할 때 필드를 정의한다. 이 유연함 덕분에 보안 분석가는 공격자의 새로운 패턴을 발견했을 때 즉시 쿼리를 짜서 대응할 수 있게 되었다. "Google for your logs"라는 별명이 붙은 이유다.

---

## 2. Splunk Enterprise Security (ES) 딥다이브

Splunk Core가 강력한 엔진이라면, Splunk ES는 그 위에 얹은 '보안 전용 완성차'다. 빅테크에서는 보통 Core의 유연함과 ES의 체계적인 프레임워크를 동시에 활용한다.

### 2.1 Notable Events: 단순 알람 그 이상
ES에서 발생하는 경보는 단순한 'Alert'이 아니라 'Notable Event'라고 불린다.
*   **상태 관리**: New, In Progress, Resolved, Closed 등 사고 대응 프로세스를 따른다.
*   **담당자 지정**: 특정 분석가에게 이벤트를 할당하고 추적할 수 있다.
*   **Incident Review**: 모든 Notable Event를 한눈에 보고 우선순위를 정하는 관제 센터의 핵심 화면이다.

### 2.2 Risk-Based Alerting (RBA): 오탐과의 전쟁
전통적인 SIEM은 임계치 기반이다. "로그인 실패 10번이면 알람을 줘." 결과는? 수천 개의 알람이 쏟아지고 분석가는 지친다(Alert Fatigue). RBA는 이 패러다임을 바꾼다.

*   **개념**: 개별 이벤트가 아니라 '엔티티(사용자, 호스트)'에 위험 점수(Risk Score)를 부여한다.
*   **시나리오**:
    1.  사용자 A가 평소 안 쓰던 IP로 로그인 (위험 점수 +5)
    2.  사용자 A가 권한 없는 파일에 접근 시도 (위험 점수 +10)
    3.  사용자 A의 계정으로 대량의 데이터 외부 전송 (위험 점수 +40)
*   **결과**: 개별 이벤트로는 알람이 안 뜨지만, 합산 점수가 50점을 넘는 순간 '고위험 Notable Event'가 생성된다. 진짜 위협을 골라내는 빅테크의 핵심 기술이다.

---

## 3. [1.5시간 워크숍] Splunk CIM 매핑 실습

Splunk ES가 제대로 작동하려면 모든 로그가 **CIM (Common Information Model)**이라는 표준 언어로 번역되어야 한다. CIM을 모르면 ES는 깡통에 불과하다.

### 3.1 왜 CIM이 필요한가?
보안 장비마다 필드 이름이 제각각이다.
*   Palo Alto Firewall: `src_ip`
*   Cisco ASA: `source_address`
*   Windows Event: `IpAddress`

CIM은 이 모든 것을 `src`라는 하나의 이름으로 통일한다. 그래야 하나의 쿼리로 모든 장비의 로그를 동시에 분석할 수 있다.

### 3.2 CIM 매핑 5단계 워크플로우

#### 1단계: 데이터 수집 (Data Input)
먼저 원천 로그를 Splunk로 가져온다. 이때 `sourcetype`을 명확히 지정하는 것이 중요하다. (예: `vendor:product:log`)

#### 2단계: 필드 추출 (Field Extraction)
로그 안에 있는 데이터를 의미 있는 필드로 쪼갠다. 정규표현식(Regex)이나 `DELIMS`를 사용한다.
```splunk
# props.conf 예시
[my_custom_firewall]
EXTRACT-src = src_ip=(?<src>\d+\.\d+\.\d+\.\d+)
EXTRACT-dest = dest_ip=(?<dest>\d+\.\d+\.\d+\.\d+)
EXTRACT-action = status=(?<action>\w+)
```

#### 3단계: 필드 에일리어싱 (Field Aliasing)
추출된 필드 이름을 CIM 표준 이름으로 연결한다.
```splunk
# props.conf 예시
FIELDALIAS-src_ip_as_src = src_ip AS src
FIELDALIAS-dest_ip_as_dest = dest_ip AS dest
```

#### 4단계: 태깅 (Tagging)
이 로그가 어떤 카테고리에 속하는지 태그를 붙인다. CIM 데이터 모델은 이 태그를 보고 데이터를 분류한다.
*   네트워크 로그라면: `tag=network`, `tag=communicate`
*   인증 로그라면: `tag=authentication`

#### 5단계: 데이터 모델 검증
`| datamodel` 명령어를 사용해 데이터가 모델에 잘 매핑되었는지 확인한다.
```splunk
| datamodel Network_Traffic search
| stats count by Network_Traffic.src, Network_Traffic.action
```

---

## 4. 상관관계 검색 (Correlation Searches) 작성

CIM 매핑이 끝났다면, 이제 진정한 SIEM의 위력인 상관관계 검색을 짤 차례다.

### 4.1 CIM 기반 검색의 장점
CIM을 사용하면 특정 벤더에 종속되지 않는 쿼리를 짤 수 있다.
```splunk
| tstats count from datamodel=Network_Traffic 
  where Network_Traffic.action=blocked 
  by Network_Traffic.src 
| where count > 100
```
이 쿼리 하나로 Palo Alto, Cisco, Checkpoint 등 사내의 모든 방화벽에서 차단된 공격지를 한 번에 찾아낼 수 있다.

### 4.2 Notable Event 생성 설정
상관관계 검색 결과가 조건에 맞으면 ES는 다음과 같은 액션을 수행한다.
1.  **Notable Event 생성**: Incident Review 대시보드에 등록.
2.  **Risk Score 부여**: 해당 IP나 사용자에게 위험 점수 가산.
3.  **외부 연동**: Slack 알림 발송이나 방화벽 IP 차단(SOAR 연동).

---

## 5. 마무리하며: 보안은 사고방식이다

보안은 도구가 아니라 사고방식의 문제다. SPL 문법을 외우는 것보다 공격자가 어떤 흔적을 남길지 상상하는 훈련을 먼저 하길 바란다. CIM은 그 상상을 현실로 만드는 언어다.

다음 주에는 이 탐지된 이벤트들을 어떻게 분석하고 대응하는지(Incident Response) 알아보자.

---

## 5주차 자가 진단 퀴즈 (심화)

**Q1. ArcSight와 같은 초기 SIEM의 'Schema-on-Write' 방식이 Splunk의 'Schema-on-Read'에 비해 가졌던 가장 큰 단점은?**
1) 데이터 저장 용량이 너무 크다.
2) 새로운 로그 소스를 추가할 때 유연성이 떨어진다.
3) 검색 속도가 Splunk보다 항상 느리다.
4) 보안 경보를 생성할 수 없다.

**Q2. Risk-Based Alerting (RBA)의 주요 목적이 아닌 것은?**
1) 알람 피로도(Alert Fatigue) 감소.
2) 개별 이벤트가 아닌 엔티티 중심의 위협 탐지.
3) 모든 로그를 실시간으로 화면에 출력하는 것.
4) 낮은 위험도의 이벤트를 조합해 고위험 위협 식별.

**Q3. CIM 매핑 과정에서 서로 다른 필드 이름(예: src_ip, source_address)을 표준 이름(src)으로 통일하는 단계는?**
1) Field Extraction
2) Field Aliasing
3) Tagging
4) Indexing

**Q4. 다음 중 네트워크 트래픽 CIM 데이터 모델에 매핑하기 위해 반드시 필요한 태그 조합은?**
1) tag=authentication
2) tag=network AND tag=communicate
3) tag=endpoint AND tag=process
4) tag=web AND tag=proxy

---

**정답 및 해설**
1.  **2) 새로운 로그 소스를 추가할 때 유연성이 떨어진다**: 데이터를 넣기 전 스키마를 정의해야 했기에 변화에 취약했다.
2.  **3) 모든 로그를 실시간으로 화면에 출력하는 것**: RBA는 의미 있는 위협을 골라내는 것이 목적이지, 모든 로그를 보여주는 것이 아니다.
3.  **2) Field Aliasing**: 에일리어싱을 통해 필드 이름을 표준화한다.
4.  **2) tag=network AND tag=communicate**: Network Traffic 데이터 모델의 기본 제약 조건이다.
