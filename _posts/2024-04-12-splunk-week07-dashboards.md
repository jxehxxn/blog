---
layout: post
title: "[Splunk 12주 완성] 7주차: 시각화와 대시보드 마스터하기"
date: 2024-04-12 09:00:00 +0900
categories: splunk
tags: [splunk, dashboard, visualization, simplexml]
---

데이터를 잘 뽑는 것과 잘 보여주는 것은 완전히 다른 영역입니다.
현업에서 30년 넘게 보안 전문가로 구르며 깨달은 사실이 하나 있죠.
아무리 훌륭한 쿼리라도 의사결정권자가 이해하지 못하면 그건 그냥 글자 뭉치일 뿐입니다.
이번 7주차에는 단순한 차트 생성을 넘어, 실질적인 비즈니스 가치를 전달하는 대시보드 구성법을 다룹니다.

### 1. C-Level vs 엔지니어: 누구를 위한 대시보드인가?
대시보드를 설계할 때 가장 먼저 던져야 할 질문은 "누가 보는가?"입니다.
C-Level 임원들은 세부적인 로그 한 줄에 관심이 없습니다.
그들은 현재 우리 조직의 보안 점수가 몇 점인지, 지난주 대비 위협이 늘었는지 줄었는지를 한눈에 보고 싶어 합니다.
반면 보안 운영 센터(SOC) 엔지니어들에게는 구체적인 IP 주소와 공격 패턴이 담긴 상세 대시보드가 필요하죠.
임원용은 요약된 단일 수치(Single Value)와 추세선 위주로, 실무자용은 필터링이 가능한 테이블과 드릴다운 기능 위주로 구성하는 게 정석입니다.

### 2. Simple XML: 대시보드의 뼈대
최근 Splunk Dashboard Studio가 화려한 UI로 주목받고 있지만, 진정한 엔지니어라면 Simple XML을 놓쳐서는 안 됩니다.
UI에서 제공하지 않는 세밀한 제어는 결국 XML 소스 코드를 직접 수정해야 가능하기 때문입니다.
`<dashboard>`, `<row>`, `<panel>`로 이어지는 계층 구조를 이해하면 복잡한 레이아웃도 자유자재로 배치할 수 있습니다.
특히 패널 간의 간격이나 조건부 서식(Conditional Formatting)을 적용할 때 XML 직접 수정은 필수적인 스킬입니다.

### 3. 토큰(Tokens)과 드릴다운(Drilldown)
대시보드를 살아 움직이게 만드는 핵심은 토큰입니다.
사용자가 드롭다운 메뉴에서 특정 서버를 선택하면, 그 값이 토큰에 담겨 대시보드 내 모든 검색 쿼리에 즉시 반영됩니다.
드릴다운은 여기서 한 발 더 나아갑니다.
파이 차트의 특정 조각을 클릭했을 때, 해당 데이터의 상세 내역을 보여주는 다른 패널로 이동하거나 검색 결과를 띄워주는 기능이죠.
이런 상호작용이 있어야만 사용자는 데이터의 흐름을 따라가며 근본 원인을 분석할 수 있습니다.

### 4. 실전 예제: 동적 대시보드 구현
아래는 파이 차트에서 특정 항목을 클릭하면 하단의 시계열 차트(Timechart)가 해당 항목으로 필터링되는 Simple XML 예시입니다.

```xml
<dashboard>
  <label>보안 이벤트 분석 대시보드</label>
  <row>
    <panel>
      <chart>
        <title>이벤트 유형별 분포 (클릭하여 필터링)</title>
        <search>
          <query>index=_internal | stats count by sourcetype</query>
          <earliest>-24h@h</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.chart">pie</option>
        <drilldown>
          <set token="selected_sourcetype">$click.value$</set>
        </drilldown>
      </chart>
    </panel>
  </row>
  <row>
    <panel depends="$selected_sourcetype$">
      <chart>
        <title>선택된 소스 타입($selected_sourcetype$)의 시간별 추이</title>
        <search>
          <query>index=_internal sourcetype="$selected_sourcetype$" | timechart count</query>
          <earliest>-24h@h</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.chart">line</option>
      </chart>
    </panel>
  </row>
</dashboard>
```
위 코드에서 `<set token="selected_sourcetype">$click.value$</set>` 부분이 핵심입니다.
사용자가 클릭한 값을 `selected_sourcetype`이라는 변수에 저장하고, 하단 패널은 이 토큰이 설정되었을 때만(`depends="$selected_sourcetype$"`) 나타나도록 설계했습니다.

### 5. 자가 진단 퀴즈 (Self-Assessment Quiz)

**문제 1:** 대시보드에서 사용자의 입력을 받아 다른 검색 쿼리에 전달하는 변수를 무엇이라고 부르는가?
**문제 2:** 특정 차트 요소를 클릭했을 때 다른 페이지로 이동하거나 동작을 수행하게 만드는 기능의 명칭은?
**문제 3:** Simple XML에서 특정 패널을 토큰 값이 존재할 때만 화면에 보이게 하려면 어떤 속성을 사용해야 하는가?

---

**정답:**
1. 토큰 (Token)
2. 드릴다운 (Drilldown)
3. depends

데이터 시각화는 단순히 예쁜 그림을 그리는 작업이 아닙니다.
복잡한 보안 로그 속에서 의미 있는 통찰을 끌어내어 의사결정을 돕는 강력한 도구임을 명심하세요.
다음 8주차에는 경보(Alerting) 시스템 구축에 대해 알아보겠습니다.
