---
layout: post
title:  "Splunk 고급편 Part 2: [Hands-on] 시계열 데이터 이상 탐지"
date:   2025-07-08 16:30:00 +0900
categories: jekyll update
---

## 시작하며: 평소와 다른 패턴을 찾아라

지난 파트에서 우리는 Splunk Machine Learning Toolkit (MLTK)을 설치하고 기본적인 인터페이스를 살펴보았습니다. 이제 MLTK의 강력한 기능 중 하나인 **시계열 데이터 이상 탐지(Time Series Anomaly Detection)**를 직접 실습해 볼 차례입니다.

시계열 데이터는 시간의 흐름에 따라 연속적으로 측정되는 데이터(예: CPU 사용률, 네트워크 트래픽, 로그인 시도 횟수, 웹사이트 방문자 수)를 의미합니다. 이러한 데이터는 주기성(요일별, 시간대별 패턴), 추세(점진적인 증가/감소), 계절성(특정 계절에 반복되는 패턴) 등 다양한 특성을 가집니다. 시계열 이상 탐지는 이러한 정상적인 패턴에서 벗어나는 '비정상적인' 움직임을 자동으로 찾아내는 기술입니다.

IT 운영 및 보안 분야에서 시계열 이상 탐지는 매우 중요합니다. 예를 들어, 평소와 다른 CPU 사용량 급증은 서비스 장애의 전조일 수 있고, 특정 시간대에 갑자기 늘어난 로그인 시도 횟수는 무차별 대입 공격(Brute-force attack)의 징후일 수 있습니다. MLTK의 `predict` 명령어를 활용하면 이러한 이상 징후를 손쉽게 탐지할 수 있습니다.

## `predict` 명령어: 미래를 예측하고 이상을 탐지하다

MLTK의 `predict` 명령어는 시계열 데이터를 분석하여 미래 값을 예측하고, 예측된 값과 실제 값의 차이를 기반으로 이상 징후를 탐지합니다. 이 명령어는 다양한 시계열 알고리즘(예: ARIMA, StateSpace, STL)을 내부적으로 사용하여 데이터의 패턴을 학습합니다.

### [Hands-on] 서버 CPU 사용량 이상 탐지

가상의 서버 CPU 사용량 데이터를 가지고 실습해 보겠습니다. Splunk에 아직 해당 데이터가 없다면, 아래 샘플 데이터를 `tutorial_logs` 인덱스에 `sourcetype=cpu_usage`로 업로드하여 사용하거나, 직접 Splunk Enterprise의 `_internal` 인덱스에 있는 `splunkd` 로그를 활용할 수도 있습니다.

**샘플 CPU 사용량 데이터 (cpu_usage.log)**

```log
2025-07-08 09:00:00 host=server01 cpu_usage=25
2025-07-08 09:01:00 host=server01 cpu_usage=27
2025-07-08 09:02:00 host=server01 cpu_usage=26
...
2025-07-08 10:00:00 host=server01 cpu_usage=95  # 이상 징후
2025-07-08 10:01:00 host=server01 cpu_usage=98  # 이상 징후
...
2025-07-08 11:00:00 host=server01 cpu_usage=30
```

**1. 데이터 준비 및 시각화**

먼저 CPU 사용량 데이터를 시간대별로 시각화하여 패턴을 확인합니다.

```spl
index=tutorial_logs sourcetype=cpu_usage host=server01
| timechart avg(cpu_usage) as avg_cpu_usage span=1m
```

이 검색을 실행하면 `server01`의 분당 평균 CPU 사용량 추이를 볼 수 있습니다. 만약 데이터에 급격한 변화가 있다면 그래프에서 시각적으로 확인할 수 있을 것입니다.

**2. `predict` 명령어를 이용한 이상 탐지**

이제 `predict` 명령어를 사용하여 CPU 사용량의 이상 징후를 탐지해 보겠습니다.

```spl
index=tutorial_logs sourcetype=cpu_usage host=server01
| timechart avg(cpu_usage) as avg_cpu_usage span=1m
| predict avg_cpu_usage as predicted_cpu_usage algorithm=StateSpace
| eval lower_bound = predicted_cpu_usage - standard_deviation * 2
| eval upper_bound = predicted_cpu_usage + standard_deviation * 2
| eval is_anomaly = if(avg_cpu_usage < lower_bound OR avg_cpu_usage > upper_bound, 1, 0)
| where is_anomaly=1
| table _time, avg_cpu_usage, predicted_cpu_usage, lower_bound, upper_bound, is_anomaly
```

**SPL 설명:**

*   `| timechart avg(cpu_usage) as avg_cpu_usage span=1m`: 1분 단위로 `avg_cpu_usage`를 계산합니다.
*   `| predict avg_cpu_usage as predicted_cpu_usage algorithm=StateSpace`: `avg_cpu_usage` 필드의 미래 값을 `StateSpace` 알고리즘으로 예측하고, 그 결과를 `predicted_cpu_usage` 필드에 저장합니다. 이 명령어는 예측값 외에도 `standard_deviation` (표준 편차) 필드를 자동으로 생성합니다.
*   `| eval lower_bound = predicted_cpu_usage - standard_deviation * 2`: 예측값의 하한선을 계산합니다. (예측값 ± 2 * 표준 편차 범위 밖을 이상 징후로 간주)
*   `| eval upper_bound = predicted_cpu_usage + standard_deviation * 2`: 예측값의 상한선을 계산합니다.
*   `| eval is_anomaly = if(avg_cpu_usage < lower_bound OR avg_cpu_usage > upper_bound, 1, 0)`: 실제 `avg_cpu_usage`가 예측 범위(`lower_bound` ~ `upper_bound`)를 벗어나면 `is_anomaly` 필드를 1로 설정합니다.
*   `| where is_anomaly=1`: 이상 징후로 탐지된 이벤트만 필터링합니다.

**3. 결과 시각화 및 해석**

이 검색을 실행한 후 **Visualization** 탭으로 이동하여 라인 차트를 선택합니다. `avg_cpu_usage`, `predicted_cpu_usage`, `lower_bound`, `upper_bound` 필드를 모두 선택하여 그래프로 그리면, 실제 CPU 사용량이 예측된 정상 범위를 벗어나는 지점(이상 징후)을 시각적으로 명확하게 확인할 수 있습니다.

## MLTK Assistant를 활용한 이상 탐지

MLTK 앱의 **Assistant** 기능을 활용하면 위와 같은 SPL을 직접 작성하지 않고도 시계열 이상 탐지 모델을 쉽게 구축할 수 있습니다.

1.  MLTK 앱으로 이동하여 **Assistant** 탭을 클릭합니다.
2.  **Anomaly Detection** 섹션에서 **Detect Numeric Outliers**를 선택합니다.
3.  **Data Source**에서 `index=tutorial_logs sourcetype=cpu_usage`를 입력합니다.
4.  **Field to Analyze**에서 `cpu_usage`를 선택합니다.
5.  **Time Series Split**에서 `host`를 선택합니다. (각 호스트별로 개별 분석)
6.  **Algorithm**에서 `StateSpace`를 선택합니다.
7.  **Train** 버튼을 클릭하여 모델을 학습시킵니다.
8.  학습이 완료되면 **Detect Anomalies** 탭에서 이상 징후가 탐지된 결과를 확인할 수 있습니다. 여기서 생성된 SPL을 복사하여 저장하거나, 대시보드 패널 또는 경보로 설정할 수 있습니다.

## 탐지된 이상 징후를 경보로 설정

이상 징후가 탐지되었을 때 자동으로 알림을 받으려면, 위에서 작성한 SPL을 경보로 설정하면 됩니다. (초급편 Part 6 참고)

*   **경보 조건:** `Number of Results is greater than 0` (이상 징후가 1건이라도 탐지되면 경보 발생)
*   **스케줄:** `Run every 5 minutes` (5분마다 이상 징후를 체크)
*   **액션:** 이메일, Slack 알림, SOAR 연동 등

## 다음 이야기

이번 파트에서는 MLTK의 `predict` 명령어를 활용하여 시계열 데이터에서 이상 징후를 탐지하는 방법을 실습했습니다. 이는 IT 운영 및 보안 분야에서 매우 유용하게 활용될 수 있는 강력한 기법입니다.

하지만 모든 데이터가 시계열 형태는 아닙니다. 다음 파트에서는 **"[Hands-on] 클러스터링을 이용한 비정상 행위 분류 (Clustering for Anomaly Detection)"**를 주제로, 비정형 로그 데이터에서 유사한 패턴을 그룹화하고, 새로운 패턴이나 소수 그룹을 이상 징후로 탐지하는 방법을 알아보겠습니다.

---
