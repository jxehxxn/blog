---
layout: post
title:  "Splunk 시리즈 Part 5: [Hands-on] Splunk 검색 언어(SPL) 첫걸음 및 기본 분석"
date:   2025-07-08 11:00:00 +0900
categories: jekyll update
---

## 시작하며: 데이터와 대화하는 언어, SPL

지금까지 우리는 Splunk를 설치하고, 데이터를 수집했으며, Source Type과 Index를 이용해 데이터를 체계적으로 정리했습니다. 이제 잘 정돈된 데이터 속에서 의미 있는 정보를 찾아낼 시간입니다. 그 핵심 도구가 바로 **SPL(Search Processing Language)**, 스플렁크 검색 언어입니다.

SPL은 단순히 특정 키워드를 찾는 것을 넘어, 방대한 데이터 속에서 원하는 것을 필터링하고, 통계를 내고, 필드를 조작하며, 결과를 시각화하는 모든 작업을 수행하는 강력한 언어입니다. 마치 데이터와 직접 '대화'하는 것과 같습니다. 이번 파트에서는 SPL의 기본 구조를 이해하고, 가장 자주 사용되는 핵심 명령어들을 실습해 보겠습니다.

## SPL의 기본 구조: 파이프(|)로 연결하는 명령어 체인

SPL은 파이프(`|`) 문자를 사용하여 여러 명령어를 체인처럼 연결하는 구조를 가지고 있습니다. 왼쪽에서 오른쪽으로 데이터가 흐르면서 각 명령어의 처리를 순서대로 거치게 됩니다.

```spl
<첫 번째 명령어: 데이터 소스 지정> | <두 번째 명령어: 필터링> | <세 번째 명령어: 통계/집계> | ...
```

*   **첫 번째 명령어**는 보통 검색할 데이터의 범위를 지정합니다. (예: `index="os_logs" sourcetype="linux_secure"`)
*   이후 **파이프(`|`)** 를 통해 전달된 데이터를 다음 명령어가 받아서 추가적인 처리를 합니다.

이러한 구조 덕분에 복잡한 분석 작업도 여러 단계의 간단한 명령어로 나누어 쉽게 수행할 수 있습니다.

## [Hands-on] 핵심 SPL 명령어 실습하기

이제 `tutorial_logs` 인덱스에 저장된 리눅스 보안 로그 샘플(`sourcetype=linux_secure`)을 가지고 실제 분석을 수행해 보겠습니다. Splunk 웹의 **Search & Reporting** 앱에서 직접 따라 해보세요.

**1. 시간 범위 설정 및 키워드 검색**

가장 먼저 해야 할 일은 분석할 시간 범위를 지정하는 것입니다. 샘플 데이터는 특정 시간에 발생했으므로, 시간 범위 선택기(Time Range Picker)를 **All time**으로 설정해야 모든 데이터가 보입니다.

*   **기본 검색: 로그인 실패(Failed password) 로그 찾아보기**

    ```spl
    index="tutorial_logs" sourcetype="linux_secure" "Failed password"
    ```

    *   `"Failed password"`: 공백이 포함된 문자열은 큰따옴표로 묶어줍니다.

**2. `stats` 명령어: 데이터 통계의 모든 것**

`stats`는 SPL에서 가장 강력하고 빈번하게 사용되는 명령어 중 하나입니다. 지정된 필드를 기준으로 데이터의 개수를 세거나, 합계, 평균 등을 계산할 수 있습니다.

*   **로그인 실패를 가장 많이 시도한 IP는?**

    ```spl
    index="tutorial_logs" sourcetype="linux_secure" "Failed password"
    | stats count by src_ip
    | sort -count
    ```

    *   `| stats count by src_ip`: `src_ip`(출발지 IP) 필드를 기준으로 이벤트의 개수(`count`)를 셉니다.
    *   `| sort -count`: `count` 필드를 기준으로 내림차순(`-`) 정렬합니다. 샘플 데이터에서는 `112.220.11.11`과 `211.45.12.33` IP가 각각 여러 번 실패한 것을 확인할 수 있습니다.

*   **시간대별 로그인 실패 추이는?**

    ```spl
    index="tutorial_logs" sourcetype="linux_secure" "Failed password"
    | timechart count
    ```

    *   `| timechart count`: 시간의 흐름에 따른 이벤트의 개수를 자동으로 계산하여 시계열 차트로 보여줍니다. 이 한 줄만으로도 언제 공격 시도가 급증했는지 한눈에 파악할 수 있습니다.

**3. `table` 명령어: 원하는 필드만 깔끔하게 보기**

검색 결과에는 수많은 필드가 있어 복잡해 보일 수 있습니다. `table` 명령어를 사용하면 내가 원하는 필드만 골라서 표 형태로 깔끔하게 정리할 수 있습니다.

*   **로그인 실패 이벤트의 시간, 출발지 IP, 시도한 계정만 보기**

    ```spl
    index="tutorial_logs" sourcetype="linux_secure" "Failed password"
    | table _time, src_ip, user
    ```

    *   `| table _time, src_ip, user`: `_time`(이벤트 시간), `src_ip`, `user` 필드만 순서대로 보여줍니다.

**4. `eval` 명령어: 새로운 필드 생성 및 조작**

`eval` 명령어는 기존 필드를 가공하거나, 여러 필드를 조합하여 새로운 필드를 동적으로 생성할 때 사용됩니다.

*   **로그인 성공 여부에 따라 'status' 필드 만들기**

    ```spl
    index="tutorial_logs" sourcetype="linux_secure"
    | eval status = if(match(_raw, "Failed password"), "실패", "성공")
    | table _time, src_ip, user, status
    ```

    *   `| eval status = if(match(_raw, "Failed password"), "실패", "성공")`: `if` 함수를 사용합니다. 전체 로그 내용(`_raw`)에 "Failed password"라는 문자열이 포함되어 있으면(`match` 함수) `status` 필드의 값을 "실패"로, 그렇지 않으면 "성공"으로 지정하는 새로운 필드를 만듭니다.

## 검색 결과를 시각화로 한눈에 파악하기

SPL의 또 다른 강력함은 검색 결과를 클릭 몇 번만으로 다양한 시각화 자료로 변환할 수 있다는 점입니다.

`stats`나 `timechart` 같은 통계 명령어를 실행하고 나면, 검색창 아래에 **Visualization** 탭이 활성화됩니다. 여기서 파이 차트, 바 차트, 라인 차트 등 원하는 형태로 시각화 방식을 변경할 수 있습니다.

예를 들어, 위에서 실행했던 '로그인 실패 시도 IP' 검색 결과를 파이 차트로 변환하면, 어떤 IP가 전체 실패의 몇 퍼센트를 차지하는지 직관적으로 파악할 수 있습니다.

## 다음 이야기

이번 파트에서는 SPL의 기본 구조와 핵심 명령어들을 맛보았습니다. `stats`, `timechart`, `table` 등의 몇 가지 명령어만 조합해도 매우 유용한 분석이 가능하다는 것을 확인하셨을 겁니다.

이제 우리는 실시간으로 데이터를 분석하고 시각화할 수 있게 되었습니다. 그렇다면 이 분석 결과를 저장해두고 계속 모니터링하거나, 특정 조건이 만족되었을 때 자동으로 알림을 받게 할 수는 없을까요?

다음 파트에서는 **"[Hands-on] 운영 효율을 극대화하는 대시보드 제작과 경보 설정"**을 주제로, 분석 결과를 자산으로 만들고 IT 운영을 자동화하는 방법을 알아보겠습니다.

---
