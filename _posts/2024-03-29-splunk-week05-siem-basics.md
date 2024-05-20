---
layout: post
title: "[Splunk 12주 완성] 5주차: SIEM의 본질과 빅테크 보안 모니터링"
date: 2024-03-29 09:00:00 +0900
categories: splunk security
---

빅테크 보안 현장에서 30년을 굴렀다.
수많은 툴이 명멸하는 걸 봤지만 SIEM은 여전히 보안 운영의 심장이다.
오늘은 단순한 툴 사용법이 아니라 빅테크 엔지니어가 어떻게 위협을 정의하고 탐지하는지 그 본질을 다뤄보자.

### SIEM은 로그 저장소가 아니다

많은 이들이 SIEM(Security Information and Event Management)을 그저 로그를 모아두는 창고로 생각한다.
큰 오산이다.
SIEM의 진짜 가치는 흩어진 점들을 연결해 선을 만드는 '맥락(Context)'에 있다.
서버 로그, 방화벽 로그, 엔드포인트 로그를 한데 모아 "이 사용자가 평소와 다른 행동을 하고 있는가?"라는 질문에 답을 내놓는 과정이다.

### Splunk Core vs Splunk Enterprise Security (ES)

Splunk Core는 강력한 엔진이다.
데이터를 수집하고 인덱싱하며 SPL로 자유롭게 검색할 수 있는 기반을 제공한다.
반면 Splunk ES는 그 엔진 위에 얹은 '보안 전용 완성차'다.
자산 관리, 위협 인텔리전스 연동, 사고 대응 워크플로우가 이미 설계되어 있다.
빅테크에서는 보통 Core의 유연함과 ES의 체계적인 프레임워크를 동시에 활용한다.

### 위협 탐지 시나리오(Use Case) 설계하기

탐지 쿼리를 짜기 전에 먼저 '무엇을 잡을 것인가'를 정의해야 한다.
이걸 Use Case 설계라고 부른다.
단순히 "로그인 실패가 많으면 알람을 줘"라고 하면 오탐(False Positive)의 늪에 빠진다.
진짜 엔지니어는 공격자의 관점에서 시나리오를 짠다.

1. **위협 정의**: 외부 공격자가 유효한 계정 정보를 얻기 위해 무차별 대입 공격(Brute Force)을 시도함.
2. **데이터 소스**: 윈도우 이벤트 로그(EventCode 4625) 또는 리눅스 auth.log.
3. **임계치 설정**: 5분 이내에 동일한 IP에서 10번 이상의 로그인 실패 발생.
4. **예외 처리**: 사내 취약점 점검 도구나 특정 자동화 스크립트 IP는 제외.

### 실전: Brute Force 탐지 SPL 작성

이제 위 시나리오를 SPL로 옮겨보자.

```splunk
index=security sourcetype="WinEventLog:Security" EventCode=4625
| stats count as failure_count, values(user) as attempted_users by src_ip
| where failure_count > 10
| lookup asset_inventory src_ip OUTPUT is_scanner
| where is_scanner != "true"
```

이 쿼리는 단순히 실패 횟수만 세지 않는다.
`src_ip`별로 어떤 사용자 계정들을 털려고 했는지(`attempted_users`)를 보여준다.
또한 `asset_inventory` 룩업을 통해 사내 스캐너 장비는 걸러낸다.
이게 바로 '맥락'이 담긴 보안 모니터링이다.

### 마무리하며

보안은 도구가 아니라 사고방식의 문제다.
SPL 문법을 외우는 것보다 공격자가 어떤 흔적을 남길지 상상하는 훈련을 먼저 하길 바란다.
다음 주에는 이 탐지된 이벤트들을 어떻게 분석하고 대응하는지(Incident Response) 알아보자.

---

### 5주차 자가 진단 퀴즈

**Q1. SIEM의 주요 기능 중 '서로 다른 소스의 로그를 연결해 의미를 찾는 과정'을 무엇이라 하는가?**
1) Indexing
2) Correlation
3) Forwarding
4) Parsing

**Q2. Splunk ES가 Splunk Core와 차별화되는 가장 큰 특징은?**
1) 데이터 수집 속도가 더 빠르다.
2) 보안 전용 데이터 모델(CIM)과 워크플로우를 제공한다.
3) 무료로 사용할 수 있다.
4) SPL을 사용하지 않아도 된다.

**Q3. Brute Force 탐지 쿼리에서 오탐을 줄이기 위해 반드시 고려해야 할 요소는?**
1) 로그의 전체 용량
2) 서버의 CPU 점유율
3) 화이트리스트(예: 사내 스캐너 IP) 처리
4) 인덱서의 개수

---

**정답 및 해설**
1. **2) Correlation**: 상관관계 분석을 통해 개별 이벤트를 하나의 위협 시나리오로 엮는다.
2. **2) 보안 전용 데이터 모델(CIM)과 워크플로우를 제공한다**: ES는 보안 운영에 최적화된 프레임워크다.
3. **3) 화이트리스트 처리**: 정상적인 자동화 도구를 제외해야 보안 관제 요원의 피로도를 줄일 수 있다.
