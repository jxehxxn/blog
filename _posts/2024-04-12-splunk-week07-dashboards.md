---
layout: post
title: "[Splunk 12주 완성] 7주차: 시각화와 대시보드 마스터하기 (심화편)"
date: 2024-04-12 09:00:00 +0900
categories: splunk
tags: [splunk, dashboard, visualization, simplexml, soc]
---

데이터를 추출하는 기술과 데이터를 시각화하는 기술은 완전히 다른 영역입니다. 30년 넘게 보안 현장에서 구르며 깨달은 진리가 하나 있습니다. 아무리 정교한 쿼리라도 의사결정권자가 5초 안에 이해하지 못하면 그건 실패한 보고서입니다. 이번 7주차 강의는 단순한 차트 생성을 넘어, 비즈니스 가치를 증명하고 실무 효율을 극대화하는 대시보드 설계의 정수를 다룹니다.

---

## 1. 대시보드의 심리학: 누구를 위한 화면인가?

대시보드를 설계하기 전, 반드시 "누가 이 화면을 보는가?"를 자문해야 합니다. 대상에 따라 정보의 밀도와 표현 방식이 완전히 달라지기 때문입니다.

### C-Level (임원진)의 관점
임원들은 세부적인 로그 한 줄에 관심이 없습니다. 그들이 원하는 것은 **'상태'**와 **'추세'**입니다.
- **핵심 질문:** "우리는 지금 안전한가?", "지난주보다 나아졌는가?", "예산 투입 대비 효과가 있는가?"
- **디자인 원칙:** 단일 수치(Single Value) 위주로 구성합니다. 빨강, 노랑, 초록의 신호등 체계를 도입하여 즉각적인 상황 판단을 돕습니다. 복잡한 테이블보다는 꺾은선 그래프로 추세를 보여주는 것이 효과적입니다.

### SOC 분석가 (실무자)의 관점
분석가들에게 대시보드는 **'작업대'**입니다. 그들은 즉시 조치할 수 있는 구체적인 데이터가 필요합니다.
- **핵심 질문:** "어떤 IP가 공격 중인가?", "어떤 사용자의 계정이 탈취되었는가?", "이상 징후의 원인은 무엇인가?"
- **디자인 원칙:** 필터링 기능이 강력해야 합니다. 드롭다운, 텍스트 입력 등을 통해 데이터를 자유자재로 쪼개볼 수 있어야 하며, 차트 클릭 시 상세 로그로 연결되는 드릴다운 기능이 필수입니다.

---

## 2. Simple XML Deep Dive: 대시보드의 뼈대

Splunk Dashboard Studio가 화려한 UI를 제공하지만, 복잡한 로직과 세밀한 제어를 위해서는 Simple XML을 반드시 이해해야 합니다.

### 계층 구조 이해
Simple XML은 다음과 같은 트리 구조를 가집니다.
1.  `<dashboard>` 또는 `<form>`: 최상위 루트 요소입니다. 입력을 받는 경우 `<form>`을 사용합니다.
2.  `<row>`: 가로 줄을 정의합니다. 하나의 로우에는 여러 개의 패널을 배치할 수 있습니다.
3.  `<panel>`: 시각화 요소가 담기는 컨테이너입니다.
4.  `<chart>`, `<table>`, `<single>`: 실제 시각화 유형을 결정합니다.
5.  `<search>`: 데이터를 가져오는 SPL 쿼리가 위치합니다.

### 레이아웃 제어
패널의 너비는 `<row>` 안의 패널 개수에 따라 자동으로 조절되지만, 특정 패널을 강조하고 싶다면 XML에서 직접 제어할 수 있습니다. 또한 `depends` 속성을 사용하면 특정 조건(토큰 설정 여부 등)이 만족될 때만 패널을 노출시킬 수 있습니다.

---

## 3. [2시간 실습] SOC 커맨드 센터 대시보드 구축

이제 실제 데이터를 사용하여 멀티 패널 대시보드를 처음부터 끝까지 구축해 보겠습니다. 모든 쿼리는 Splunk 내부 인덱스(`_internal`, `_audit`)를 사용하므로 별도의 데이터 로드가 필요 없습니다.

### 단계 1: 대시보드 기본 설정 및 입력 필드 추가
먼저 시간 범위와 인덱스를 선택할 수 있는 입력 필드를 만듭니다.

```xml
<form>
  <label>SOC 커맨드 센터 (실습용)</label>
  <fieldset submitButton="false">
    <input type="time" token="time_tok">
      <label>시간 범위 선택</label>
      <default>
        <earliest>-24h@h</earliest>
        <latest>now</latest>
      </default>
    </input>
    <input type="dropdown" token="index_tok">
      <label>인덱스 선택</label>
      <choice value="_internal">Internal</choice>
      <choice value="_audit">Audit</choice>
      <default>_internal</default>
    </input>
  </fieldset>
```

### 단계 2: 요약 정보 패널 (Single Value)
상단에는 전체 이벤트 수와 에러 발생 건수를 배치합니다.

```xml
  <row>
    <panel>
      <single>
        <title>전체 이벤트 수</title>
        <search>
          <query>index=$index_tok$ | stats count</query>
          <earliest>$time_tok.earliest$</earliest>
          <latest>$time_tok.latest$</latest>
        </search>
        <option name="drilldown">none</option>
        <option name="refresh.display">progressbar</option>
      </single>
    </panel>
    <panel>
      <single>
        <title>에러/경고 발생 건수</title>
        <search>
          <query>index=$index_tok$ log_level IN (ERROR, WARN) | stats count</query>
          <earliest>$time_tok.earliest$</earliest>
          <latest>$time_tok.latest$</latest>
        </search>
        <option name="rangeColors">["0x53a051","0x0877a6","0xf8be34","0xf1592a","0xd93f3c"]</option>
        <option name="rangeValues">[0,10,50,100]</option>
        <option name="useColors">1</option>
      </single>
    </panel>
  </row>
```

### 단계 3: 시각화 분석 패널 (Pie & Bar Chart)
데이터의 분포를 한눈에 파악할 수 있는 차트를 추가합니다.

```xml
  <row>
    <panel>
      <chart>
        <title>소스타입별 분포</title>
        <search>
          <query>index=$index_tok$ | top limit=10 sourcetype</query>
          <earliest>$time_tok.earliest$</earliest>
          <latest>$time_tok.latest$</latest>
        </search>
        <option name="charting.chart">pie</option>
        <drilldown>
          <set token="selected_st">$click.value$</set>
        </drilldown>
      </chart>
    </panel>
    <panel>
      <chart>
        <title>시간대별 로그 레벨 추이</title>
        <search>
          <query>index=$index_tok$ | timechart count by log_level</query>
          <earliest>$time_tok.earliest$</earliest>
          <latest>$time_tok.latest$</latest>
        </search>
        <option name="charting.chart">column</option>
        <option name="charting.chart.stackMode">stacked</option>
      </chart>
    </panel>
  </row>
```

### 단계 4: 동적 상세 테이블 (Token Passing & Drilldown)
파이 차트에서 특정 소스타입을 클릭하면 해당 데이터만 보여주는 상세 테이블을 하단에 배치합니다.

```xml
  <row>
    <panel depends="$selected_st$">
      <table>
        <title>상세 로그 분석: $selected_st$</title>
        <search>
          <query>index=$index_tok$ sourcetype="$selected_st$" | table _time, component, log_level, message | sort - _time</query>
          <earliest>$time_tok.earliest$</earliest>
          <latest>$time_tok.latest$</latest>
        </search>
        <option name="count">10</option>
        <option name="drilldown">cell</option>
        <drilldown>
          <link target="_blank">search?q=index=$index_tok$ sourcetype="$selected_st$" component=$row.component$</link>
        </drilldown>
      </table>
      <html>
        <button class="btn btn-primary" id="btn_clear">필터 초기화</button>
      </html>
    </panel>
  </row>
</form>
```

---

## 4. 고급 상호작용: 토큰과 드릴다운 마스터하기

대시보드를 단순한 그림에서 강력한 분석 도구로 바꾸는 핵심 기술입니다.

### 토큰 전달 (Token Passing)
위 실습 코드에서 `<set token="selected_st">$click.value$</set>` 부분이 핵심입니다. 사용자가 차트의 특정 요소를 클릭하면 그 값이 `selected_st`라는 변수에 저장됩니다. 이 변수는 다른 패널의 쿼리에서 `$selected_st$` 형태로 참조되어 동적 필터링을 수행합니다.

### 계층형 입력 (Cascading Inputs)
인덱스를 선택하면 해당 인덱스에 존재하는 소스타입만 드롭다운에 나타나게 하고 싶을 때 사용합니다.
1.  첫 번째 입력(인덱스)에서 토큰 `idx_tok`을 설정합니다.
2.  두 번째 입력(소스타입)의 검색 쿼리에 `index=$idx_tok$ | stats count by sourcetype`을 사용합니다.
이렇게 하면 첫 번째 선택에 따라 두 번째 선택지가 실시간으로 변합니다.

### 드릴다운 전략
- **내부 드릴다운:** 동일 대시보드 내의 다른 패널을 갱신하거나 숨겨진 패널을 보여줍니다.
- **외부 드릴다운:** 클릭 시 새로운 탭에서 검색 창을 열거나, 외부 티켓팅 시스템(Jira, ServiceNow)으로 데이터를 전달합니다.

---

## 5. 실전 SPL 레퍼런스 (대시보드용)

대시보드 패널을 구성할 때 자주 사용하는 패턴들입니다.

1.  **상태 표시 (Single Value):**
    `index=_internal | stats count as total | eval status=if(total > 1000, "High", "Normal")`
2.  **비율 계산 (Gauge/Pie):**
    `index=_internal | stats count(eval(log_level="ERROR")) as errors, count as total | eval error_rate=(errors/total)*100 | table error_rate`
3.  **이상 징후 탐지 (Timechart):**
    `index=_internal | timechart count as actual | predict actual as predicted`

---

## 6. 마무리: 훌륭한 대시보드를 위한 체크리스트

1.  **5초 룰:** 사용자가 화면을 보고 5초 안에 상황을 파악할 수 있는가?
2.  **색상 절제:** 너무 많은 색상은 인지 부하를 줍니다. 의미 있는 차이에만 색상을 사용하세요.
3.  **일관성:** 모든 패널의 시간 범위가 동일하게 연동되는가?
4.  **성능:** 대시보드 로딩에 10초 이상 걸린다면 쿼리 최적화나 요약 인덱싱(Summary Indexing)을 검토하세요.

데이터 시각화는 예술이 아니라 공학입니다. 사용자의 의사결정 과정을 설계한다는 마음가짐으로 대시보드에 접근하시기 바랍니다. 다음 8주차에는 실시간 탐지와 경보(Alerting) 시스템 구축을 다룹니다.
