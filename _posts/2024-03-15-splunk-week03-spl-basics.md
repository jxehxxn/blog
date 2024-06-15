---
layout: post
title: "Splunk 12주 완성: 3주차: SPL(Search Processing Language) 기초 (3시간 강의용)"
date: 2024-03-15 09:00:00 +0900
categories: splunk security
---

# [강의 교안] Splunk 12주 완성: 3주차: SPL 기초와 데이터 파이프라인의 이해

**강의 시간**: 3시간 (이론 1.5시간, 실습 1.5시간)
**대상**: 보안 분석가 및 시스템 운영자
**목표**: SPL의 작동 원리를 이해하고, 대규모 로그 데이터를 효율적으로 처리하는 능력을 배양한다.

---

## 1교시: SPL의 철학과 MapReduce 아키텍처 (45분)

### 1.1 SPL이란 무엇인가?
SPL(Search Processing Language)은 단순히 데이터를 검색하는 언어가 아닙니다. 이는 **데이터 파이프라인 처리 언어**입니다. 유닉스 쉘의 파이프(`|`) 개념을 도입하여, 왼쪽에서 오른쪽으로 데이터를 흐르게 하며 각 단계에서 변환, 필터링, 집계를 수행합니다.

### 1.2 Splunk의 분산 검색 아키텍처 (MapReduce)
스플렁크의 검색은 구글의 MapReduce 프레임워크와 유사하게 작동합니다.

1.  **Search Head (SH)**: 사용자의 검색 요청을 받아 구문을 분석하고, 이를 모든 인덱서(Indexer)에게 전달합니다.
2.  **Indexer (IDX)**: 자신이 보유한 로컬 디스크(Buckets)에서 데이터를 검색합니다. (Map 단계)
3.  **Aggregation**: 인덱서들은 검색된 결과를 Search Head로 다시 보냅니다.
4.  **Final Result**: Search Head는 여러 인덱서로부터 받은 데이터를 최종적으로 합치고 정렬하여 사용자에게 보여줍니다. (Reduce 단계)

### 1.3 명령어의 종류와 자원 소모 (CPU/Memory)
검색 성능을 최적화하려면 명령어의 특성을 알아야 합니다.

*   **Streaming Commands (스트리밍 명령어)**:
    *   **예시**: `search`, `eval`, `where`, `rex`, `fields`
    *   **특징**: 각 이벤트를 개별적으로 처리합니다. 인덱서에서 병렬로 실행되므로 매우 빠릅니다.
    *   **비용**: 메모리 소모가 적고 CPU를 효율적으로 사용합니다.
*   **Transforming Commands (변환 명령어)**:
    *   **예시**: `stats`, `chart`, `timechart`, `top`
    *   **특징**: 데이터를 표(Table) 형태로 변환합니다. 인덱서에서 중간 집계를 하고 Search Head에서 최종 집계를 합니다.
    *   **비용**: Search Head의 메모리를 많이 사용하며, 데이터가 많을수록 처리 시간이 길어집니다.
*   **Non-Streaming Commands (비스트리밍 명령어)**:
    *   **예시**: `sort`, `dedup`
    *   **특징**: 전체 데이터셋을 확인해야 결과를 낼 수 있습니다.
    *   **비용**: **가장 위험한 명령어**입니다. 모든 데이터를 Search Head의 메모리에 올려야 하므로, 대량의 데이터에서 `sort`를 잘못 쓰면 검색이 실패하거나 시스템이 느려집니다.

---

## 2교시: SPL 진화 과정: 10단계 실전 예제 (45분)

단순한 검색에서 복잡한 분석으로 나아가는 과정을 살펴봅니다.

### 단계 1: 기본 검색 (Disk I/O 발생)
```splunk
index=web_logs
```
*   **설명**: `web_logs` 인덱스의 모든 데이터를 가져옵니다.
*   **비용**: 매우 높음. 필터링 없이 모든 디스크 블록을 읽습니다.

### 단계 2: 키워드 필터링 (Indexer-side Filtering)
```splunk
index=web_logs status=404
```
*   **설명**: 404 에러 로그만 골라냅니다.
*   **비용**: 낮음. 인덱서가 디스크에서 필요한 데이터만 추출하여 메모리로 올립니다.

### 단계 3: 정규표현식을 이용한 필드 추출 (Streaming)
```splunk
index=web_logs status=404 
| rex field=_raw "GET (?<endpoint>\S+)"
```
*   **설명**: 원본 로그(`_raw`)에서 요청 경로를 추출하여 `endpoint` 필드를 만듭니다.
*   **비용**: 중간. CPU를 사용하여 텍스트를 파싱하지만, 인덱서에서 병렬로 처리됩니다.

### 단계 4: 조건문 활용 (Streaming)
```splunk
index=web_logs status=404 
| rex field=_raw "GET (?<endpoint>\S+)"
| eval is_api=if(like(endpoint, "/api/%"), "API", "Web")
```
*   **설명**: 경로가 `/api/`로 시작하면 "API", 아니면 "Web"으로 분류합니다.
*   **비용**: 낮음. 단순한 논리 연산입니다.

### 단계 5: 기본 통계 (Transforming)
```splunk
index=web_logs status=404 
| rex field=_raw "GET (?<endpoint>\S+)"
| eval is_api=if(like(endpoint, "/api/%"), "API", "Web")
| stats count by endpoint, is_api
```
*   **설명**: 엔드포인트별 발생 횟수를 집계합니다.
*   **비용**: 중간. Search Head에서 데이터를 합치는 과정이 필요합니다.

### 단계 6: 데이터 정렬 (Non-Streaming)
```splunk
... | stats count by endpoint, is_api 
| sort - count
```
*   **설명**: 가장 많이 발생한 순서대로 정렬합니다.
*   **비용**: 높음. 집계된 결과를 메모리에 모두 올려 정렬해야 합니다.

### 단계 7: 결과 필터링 (Post-aggregation Filtering)
```splunk
... | stats count by endpoint, is_api 
| where count > 100
```
*   **설명**: 100번 이상 발생한 것만 남깁니다.
*   **비용**: 낮음. 이미 줄어든 데이터셋에서 필터링하므로 빠릅니다.

### 단계 8: 외부 데이터 결합 (Lookup)
```splunk
... | lookup dns_lookup ip as clientip OUTPUT hostname
```
*   **설명**: IP 주소를 호스트네임으로 변환합니다.
*   **비용**: 중간. 외부 파일(CSV 등)을 참조하므로 I/O가 발생합니다.

### 단계 9: 시계열 분석 (Time-based Analysis)
```splunk
index=web_logs status=404 
| timechart span=1h count by is_api
```
*   **설명**: 1시간 단위로 API와 Web의 에러 발생 추이를 봅니다.
*   **비용**: 중간. 시간축을 기준으로 데이터를 버킷팅합니다.

### 단계 10: 고급 통계 (Distinct Count)
```splunk
index=web_logs 
| stats dc(clientip) as unique_visitors by endpoint 
| where unique_visitors > 50
```
*   **설명**: 특정 페이지를 방문한 고유 방문자(IP) 수를 계산합니다.
*   **비용**: 높음. 고유 값을 추적하기 위해 메모리에 IP 목록을 유지해야 합니다.

---

## 3교시: 실습: Apache 및 Windows 로그 통합 분석 (1.5시간)

### [실습 시나리오]
우리 회사의 웹 서버에 비정상적인 접근이 감지되었습니다. 공격자는 웹 취약점을 스캔한 후, 탈취한 계정으로 내부 서버에 로그인한 것으로 의심됩니다. Apache 로그와 Windows Event 로그를 연동하여 공격의 전말을 밝히십시오.

### 실습 1: 웹 스캐닝 행위 분석 (30분)
1.  **목표**: 공격자의 IP를 식별하고 사용된 도구를 파악합니다.
2.  **과제**:
    *   `index=web`에서 `status=404`가 가장 많이 발생하는 IP를 찾으세요.
    *   `rex`를 사용하여 `User-Agent` 필드를 추출하세요.
    *   `stats`를 사용하여 IP별로 사용된 `User-Agent`의 종류를 나열하세요.
    *   **힌트**: `stats values(user_agent) by clientip`

### 실습 2: 무차별 대입 공격(Brute Force) 탐지 (30분)
1.  **목표**: 로그인 실패가 빈번한 계정을 찾습니다.
2.  **과제**:
    *   Windows Event 로그(`index=win_logs`)에서 Event ID `4625`(로그인 실패)를 필터링하세요.
    *   `timechart`를 사용하여 5분 단위로 실패 횟수를 시각화하세요.
    *   특정 시간대에 실패 횟수가 급증한 계정(`TargetUserName`)을 식별하세요.

### 실습 3: 공격 경로 추적 (Lateral Movement) (30분)
1.  **목표**: 웹 공격 IP와 성공한 로그인 IP를 매칭합니다.
2.  **과제**:
    *   실습 1에서 찾은 공격 IP가 Windows 서버에 성공적으로 로그인(Event ID `4624`)한 기록이 있는지 검색하세요.
    *   로그인에 성공했다면, 해당 계정으로 어떤 프로세스를 실행했는지 `index=win_logs EventCode=4688`을 통해 확인하세요.
    *   최종적으로 공격자의 IP, 사용된 계정, 실행된 악성 프로세스명을 리포트로 작성하세요.

---

## 강의 마무리 및 Q&A (15분)

*   **핵심 요약**:
    1.  SPL은 파이프라인 구조다.
    2.  필터링(`search`)은 최대한 앞에 배치하여 디스크 읽기를 줄여라.
    3.  `sort`, `dedup` 같은 비스트리밍 명령어는 신중하게 사용하라.
    4.  데이터의 흐름(MapReduce)을 이해하면 성능 최적화가 보인다.

*   **다음 주 예고**: 4주차: 고급 통계와 데이터 모델링 (Data Model & Pivot)
