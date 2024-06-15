---
layout: post
title: "[Splunk 12주 코스] 4주차: 고급 SPL과 데이터 정제 (Advanced SPL & Data Refinement) - 마스터 클래스"
date: 2024-03-22 09:00:00 +0900
categories: splunk
---

# 제 4장: 고급 SPL을 활용한 대규모 데이터 분석 및 쿼리 최적화

빅테크 보안 현장에서 수조 개의 이벤트를 다루다 보면 쿼리 하나가 전체 클러스터 성능을 좌우하는 상황을 자주 마주합니다. 단순히 결과를 뽑아내는 단계를 넘어, 수십억 건의 데이터셋에서도 수 초 내에 결과를 반환하는 '성능 최적화된 쿼리'를 작성하는 능력이 전문가와 초보자를 가르는 결정적인 기준이 됩니다.

이번 장에서는 스플렁크의 가장 강력하면서도 위험한 명령어인 `join`, `subsearch`, `transaction`을 깊게 파헤치고, 이를 `stats`와 `lookup`으로 대체하여 성능을 극대화하는 전략을 다룹니다. 또한, 데이터 모델(Data Models)과 CIM(Common Information Model)을 활용한 표준화된 분석 환경 구축 방법까지 상세히 살펴보겠습니다.

---

## 1. join과 서브서치(Subsearch)의 치명적인 함정

많은 입문자가 SQL의 익숙함 때문에 가장 먼저 찾는 명령어가 `join`입니다. 하지만 스플렁크 아키텍처에서 `join`은 성능의 적이자, 데이터 부정확성의 주범이 될 수 있습니다.

### 1.1 join의 작동 원리와 한계
스플렁크에서 `join`은 내부적으로 **서브서치(Subsearch)** 메커니즘을 사용합니다. 쿼리가 실행되면 스플렁크는 대괄호 `[]` 안에 있는 서브서치를 먼저 실행하고, 그 결과를 메모리에 적재한 뒤 메인 서브서치의 결과와 비교하여 결합합니다. 이 과정에서 다음과 같은 치명적인 제약 사항이 발생합니다.

1.  **10,000건의 결과 제한 (The 10k Limit):**
    스플렁크의 `limits.conf` 설정에 따라 서브서치가 반환할 수 있는 결과는 기본적으로 **10,000건**으로 제한됩니다. 만약 서브서치 대상 데이터가 100,000건이라 하더라도, 스플렁크는 아무런 경고 없이 상위 10,000건만 가져와 `join`을 수행합니다. 나머지 90,000건에 대한 데이터는 유실되며, 이는 분석 결과의 심각한 왜곡을 초래합니다.
2.  **메모리 타임아웃 (maxtime):**
    서브서치는 기본적으로 60초(`maxtime = 60`) 이내에 완료되어야 합니다. 대규모 인덱스에서 복잡한 연산을 수행하는 서브서치가 60초를 넘기면, 스플렁크는 해당 서브서치를 강제로 종료하고 그때까지 수집된 부분적인 결과만 사용합니다.
3.  **단일 스레드 처리:**
    `join`은 분산 검색 환경에서도 서브서치 결과를 검색 헤드(Search Head)의 메모리에 모아서 처리해야 하므로, 인덱서의 병렬 처리 능력을 제대로 활용하지 못합니다.

### 1.2 서브서치 제한 우회 전략
서브서치의 한계를 극복하기 위해 다음과 같은 방법을 고려해야 합니다.

*   **`stats`를 활용한 "OR" 패턴 (권장):** 두 인덱스의 데이터를 한 번에 불러와 `stats`로 묶는 방식입니다. (자세한 내용은 2절에서 다룹니다.)
*   **`append` 명령어:** 서브서치 결과를 메인 결과 뒤에 붙이는 방식이지만, 이 역시 `maxresultrows` 제한(기본 50,000건)이 존재합니다.
*   **`map` 명령어:** 서브서치 결과의 각 행에 대해 메인 검색을 반복 실행합니다. 제한은 없지만 성능이 매우 느려 극히 제한적인 상황에서만 사용합니다.

---

## 2. join vs stats: 1시간 분량의 심층 비교 강의

실무에서 가장 중요한 기술은 `join`을 `stats`로 리팩토링하는 능력입니다. 이를 "The Power of Stats"라고 부르기도 합니다.

### 2.1 왜 stats가 더 빠른가?
`stats`는 스플렁크의 분산 검색 아키텍처(Map-Reduce 방식)를 완벽하게 활용합니다. 각 인덱서(Indexer)에서 데이터를 부분적으로 집계(Map)한 뒤, 검색 헤드에서 최종 결과만 합치기(Reduce) 때문에 네트워크 트래픽과 메모리 사용량을 획기적으로 줄일 수 있습니다.

### 2.2 리팩토링 예시: 사용자 정보 결합
**[Bad Case: join 사용]**
```spl
index=web_logs
| join user_id [search index=user_info | fields user_id, department, region]
| stats count by department
```
*   문제점: `user_info` 인덱스의 사용자가 1만 명을 넘으면 데이터가 누락됨. 검색 헤드 메모리 부하 가중.

**[Good Case: stats 사용]**
```spl
(index=web_logs) OR (index=user_info)
| stats values(department) as department, values(region) as region, count(eval(index="web_logs")) as web_count by user_id
| where web_count > 0
| stats sum(web_count) by department
```
*   장점: 10,000건 제한 없음. 모든 처리가 인덱서에서 병렬로 수행됨. 데이터 유실 가능성 제로.

---

## 3. transaction vs stats: 세션 분석의 정석

`transaction` 명령어는 이벤트 간의 시간적 선후 관계나 특정 필드(예: session_id)를 기준으로 이벤트를 묶어주는 매우 편리한 도구입니다. 하지만 대규모 환경에서는 '성능 킬러'로 돌변합니다.

### 3.1 transaction의 비용
`transaction`은 모든 관련 이벤트를 검색 헤드의 메모리에 원본 그대로 유지해야 합니다. 수백만 건의 세션을 처리할 경우 검색 헤드의 RAM이 고갈되어 검색이 실패하거나 클러스터 전체가 느려질 수 있습니다.

### 3.2 stats를 이용한 세션화 (Sessionization)
대부분의 세션 분석은 `stats`로 대체 가능하며, 속도는 수십 배 더 빠릅니다.

**[transaction 방식]**
```spl
index=access_logs
| transaction session_id maxpause=30m
| table session_id, duration, eventcount
```

**[stats 방식]**
```spl
index=access_logs
| stats min(_time) as start_time, max(_time) as end_time, count as eventcount by session_id
| eval duration = end_time - start_time
| table session_id, duration, eventcount
```
*   `stats` 방식은 원본 이벤트를 메모리에 들고 있지 않고, 집계된 값(시작 시간, 종료 시간, 개수)만 유지하므로 메모리 효율성이 압도적입니다.

---

## 4. 실전 시나리오: 느린 쿼리 리팩토링 (Case Study)

### 상황 설명
보안 분석가인 당신은 특정 IP(`src_ip`)가 방화벽에서 차단된 내역과 해당 IP의 DHCP 할당 기록(호스트명 확인)을 결합하여 보고서를 작성해야 합니다. 방화벽 로그는 하루에 10억 건, DHCP 로그는 1,000만 건이 발생합니다.

### 4.1 문제의 느린 쿼리 (The "Slow" Query)
```spl
index=firewall action=blocked
| join src_ip [
    search index=dhcp
    | dedup src_ip sortby -_time
    | fields src_ip, hostname
]
| stats count by hostname
```
**분석:**
1.  `join` 내부의 서브서치가 `index=dhcp` 전체를 뒤져야 하므로 매우 느립니다.
2.  DHCP 할당 기록이 1만 건을 넘으면 일부 IP의 호스트명을 찾지 못합니다.
3.  방화벽 로그 10억 건을 먼저 필터링한 뒤 `join`을 수행하므로, `join` 연산 자체가 병목이 됩니다.

### 4.2 최적화된 쿼리 (The "Fast" Query)
```spl
(index=firewall action=blocked) OR (index=dhcp)
| stats values(hostname) as hostname, count(eval(index="firewall")) as block_count by src_ip
| where block_count > 0 AND isnotnull(hostname)
| stats sum(block_count) by hostname
```
**최적화 포인트:**
1.  **Single Pass Search:** 인덱스를 한 번만 스캔하여 필요한 모든 데이터를 가져옵니다.
2.  **No Subsearch Limits:** `stats`를 사용하여 1만 건 제한을 완전히 제거했습니다.
3.  **Parallel Processing:** 인덱서들이 각자 맡은 데이터 조각에서 `src_ip`별로 집계를 수행하므로 검색 속도가 비약적으로 향상됩니다.

---

## 5. 데이터 모델(Data Models)과 CIM의 힘

빅테크 규모의 환경에서는 원본 로그를 직접 검색하는 것보다 가속화된 데이터 모델을 사용하는 것이 표준입니다.

### 5.1 데이터 모델 가속화 (Acceleration)
데이터 모델 가속화를 설정하면 스플렁크는 백그라운드에서 **TSIDX(Time-Series Index)** 파일을 생성합니다. 이는 원본 데이터의 요약본으로, 검색 시 원본 로그를 읽지 않고 이 요약 파일만 읽기 때문에 검색 속도가 100배 이상 빨라질 수 있습니다.

### 5.2 tstats 명령어
가속화된 데이터 모델을 검색할 때는 `tstats` 명령어를 사용합니다. 이는 스플렁크에서 가장 빠른 검색 명령어입니다.
```spl
| tstats count from datamodel=Network_Traffic where All_Traffic.action=blocked by All_Traffic.src_ip
```

### 5.3 CIM (Common Information Model)
서로 다른 제조사의 장비(Cisco, Palo Alto, Fortinet 등)에서 발생하는 로그는 필드명이 제각각입니다. CIM은 이를 `src`, `dest`, `action` 등 표준화된 이름으로 매핑합니다. 이를 통해 분석가는 장비 종류에 상관없이 동일한 쿼리로 분석을 수행할 수 있습니다.

---

## 6. 전문가를 위한 쿼리 작성 골든 룰 (Golden Rules)

1.  **Filter Early:** `search`나 `where` 절을 최대한 앞쪽에 배치하여 처리할 데이터 양을 줄이세요.
2.  **Fields Command:** 필요한 필드만 명시적으로 선택(`| fields src_ip, dest_ip`)하여 메모리 사용량을 최소화하세요.
3.  **Avoid Join/Transaction:** 99%의 경우 `stats`나 `lookup`으로 대체 가능합니다.
4.  **Use Lookups for Metadata:** 정적인 정보(부서명, 자산 정보 등)는 `join` 대신 `lookup`을 사용하세요.
5.  **Check Job Inspector:** 쿼리 실행 후 'Job Inspector'를 열어 어떤 단계에서 시간이 많이 소요되는지 반드시 확인하는 습관을 들이세요.

---

### 자가 진단 및 심화 과제
1.  `join` 명령어의 기본 결과 제한 건수는 몇 건이며, 이를 초과할 경우 어떤 현상이 발생하는가?
2.  `transaction` 대신 `stats`를 사용하여 세션을 묶을 때의 장점 세 가지를 서술하시오.
3.  수십억 건의 데이터에서 특정 필드의 통계를 1초 내에 뽑아야 한다면 어떤 기술(명령어/기능)을 사용해야 하는가?

**정답 가이드:**
1. 10,000건. 경고 없이 초과 데이터가 무시되어 결과가 왜곡됨.
2. 메모리 효율성, 분산 처리 가능(속도 향상), 결과 제한 없음.
3. 데이터 모델 가속화와 `tstats` 명령어.

---
현업에서는 단순히 돌아가는 쿼리가 아니라, 시스템에 부담을 주지 않으면서도 정확한 결과를 '가장 빠르게' 내놓는 쿼리가 정답입니다. 이번 장에서 배운 `stats` 중심의 사고방식을 체득한다면, 여러분은 이미 상위 1%의 스플렁크 전문가입니다.
