---
layout: post
title:  "Splunk 고급편 Part 3: [Hands-on] 클러스터링을 이용한 비정상 행위 분류"
date:   2025-07-08 17:00:00 +0900
categories: jekyll update
---

## 시작하며: 패턴 속에서 숨겨진 이상을 찾다

지난 파트에서는 시계열 데이터의 정상 범위를 예측하고 벗어나는 이상 징후를 탐지하는 방법을 배웠습니다. 하지만 모든 데이터가 시간의 흐름에 따라 명확한 패턴을 보이는 것은 아닙니다. 특히 로그 메시지나 프로세스 실행 정보처럼 비정형(Unstructured) 또는 반정형(Semi-structured) 데이터에서는 '평소와 다른' 행위를 찾아내는 것이 더욱 어렵습니다.

이러한 경우에 유용하게 활용할 수 있는 머신러닝 기법이 바로 **클러스터링(Clustering)**입니다. 클러스터링은 데이터 포인트들을 서로의 유사성에 기반하여 그룹(클러스터)으로 묶는 비지도 학습(Unsupervised Learning)의 한 종류입니다. 즉, 미리 정답(레이블)을 알려주지 않아도 데이터 스스로 패턴을 찾아 그룹을 형성합니다.

보안 분석에서는 클러스터링을 통해 다음과 같은 이상 징후를 탐지할 수 있습니다.

*   **새로운 패턴:** 기존에 없던 새로운 클러스터에 속하는 데이터 (예: 이전에 보지 못했던 프로세스 실행 패턴)
*   **소수 클러스터:** 매우 적은 수의 데이터만 포함하는 클러스터 (예: 극소수의 사용자만 수행하는 비정상적인 로그인 시도)
*   **클러스터 이탈:** 기존 클러스터에 속해야 할 데이터가 갑자기 다른 클러스터로 이동하는 경우

이번 파트에서는 MLTK의 클러스터링 알고리즘을 활용하여 비정형 로그 데이터에서 비정상 행위를 분류하고 탐지하는 방법을 실습해 보겠습니다.

## 클러스터링 알고리즘: K-Means와 DBSCAN

MLTK는 다양한 클러스터링 알고리즘을 제공합니다. 그중 대표적인 두 가지를 소개합니다.

*   **K-Means:** 데이터를 미리 정해진 K개의 클러스터로 나눕니다. 각 데이터 포인트는 가장 가까운 클러스터의 중심(Centroid)에 할당됩니다. 클러스터의 개수(K)를 미리 지정해야 한다는 단점이 있지만, 계산이 빠르고 직관적입니다.
*   **DBSCAN (Density-Based Spatial Clustering of Applications with Noise):** 밀도 기반 클러스터링 알고리즘으로, 데이터 포인트의 밀도를 기반으로 클러스터를 형성합니다. 클러스터의 개수를 미리 지정할 필요가 없으며, 노이즈(Noise)를 이상 징후로 분류할 수 있다는 장점이 있습니다.

### [Hands-on] 로그인 실패 로그 클러스터링을 통한 이상 탐지

Splunk 초급편에서 사용했던 `tutorial_logs` 인덱스의 `linux_secure` 로그 중, 로그인 실패(`Failed password`) 이벤트를 클러스터링하여 비정상적인 로그인 시도 패턴을 찾아보겠습니다.

**1. 데이터 준비 및 필드 추출**

클러스터링을 위해서는 분석할 필드들을 준비해야 합니다. 로그인 실패 로그에서 `src_ip`, `user`, `authentication_method` (예: ssh2, password) 등의 필드를 추출합니다.

```spl
index=tutorial_logs sourcetype=linux_secure "Failed password"
| rex "from (?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) port \d+ (?<authentication_method>\w+)"
| rex "for (invalid user )?(?<user>\w+) from"
| table _time, src_ip, user, authentication_method, _raw
```

*   `rex`: 정규 표현식(Regular Expression)을 사용하여 `_raw` 로그에서 `src_ip`, `user`, `authentication_method` 필드를 추출합니다. (샘플 로그에 맞춰 조정)

**2. `kmeans`를 이용한 클러스터링**

이제 추출된 필드를 기반으로 `kmeans` 알고리즘을 사용하여 클러스터링을 수행합니다. 여기서는 `src_ip`와 `user` 필드를 클러스터링의 기준으로 사용하고, 클러스터 개수(k)를 3으로 지정해 보겠습니다.

```spl
index=tutorial_logs sourcetype=linux_secure "Failed password"
| rex "from (?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) port \d+ (?<authentication_method>\w+)"
| rex "for (invalid user )?(?<user>\w+) from"
| fields src_ip, user, authentication_method, _raw
| fit Kmeans k=3 src_ip, user
| table _time, src_ip, user, authentication_method, cluster
```

**SPL 설명:**

*   `| fields src_ip, user, authentication_method, _raw`: 클러스터링에 사용할 필드와 원본 로그를 선택합니다.
*   `| fit Kmeans k=3 src_ip, user`: `src_ip`와 `user` 필드를 기준으로 데이터를 3개의 클러스터로 나눕니다. `fit` 명령어는 모델을 학습시키고, 각 이벤트에 `cluster` 필드(0, 1, 2)를 할당합니다.

**결과 해석:**
검색 결과에서 `cluster` 필드를 확인하면, 유사한 `src_ip`와 `user` 조합을 가진 로그인 실패 이벤트들이 동일한 클러스터에 할당된 것을 볼 수 있습니다. 예를 들어, 특정 IP에서 `admin` 계정으로 반복적으로 로그인 실패가 발생하는 패턴이 하나의 클러스터로 묶일 수 있습니다.

**3. `DBSCAN`을 이용한 클러스터링 및 노이즈 탐지**

`DBSCAN`은 클러스터의 개수를 미리 지정할 필요가 없으며, 밀도가 낮은 영역의 데이터(노이즈)를 이상 징후로 분류할 수 있다는 장점이 있습니다. `DBSCAN`은 숫자형 필드에 더 적합하므로, `src_ip`를 숫자형으로 변환하여 사용해 보겠습니다.

```spl
index=tutorial_logs sourcetype=linux_secure "Failed password"
| rex "from (?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) port \d+ (?<authentication_method>\w+)"
| rex "for (invalid user )?(?<user>\w+) from"
| eval src_ip_numeric = cidrmatch("0.0.0.0/0", src_ip)
| fields src_ip_numeric, user, authentication_method, _raw
| fit DBSCAN eps=0.5 min_samples=2 src_ip_numeric
| table _time, src_ip, user, authentication_method, cluster
| where cluster = -1  // -1은 노이즈(이상 징후) 클러스터를 의미
```

**SPL 설명:**

*   `| eval src_ip_numeric = cidrmatch("0.0.0.0/0", src_ip)`: `src_ip`를 숫자형으로 변환합니다. (실제 IP 주소를 숫자로 변환하는 더 복잡한 로직이 필요할 수 있습니다.)
*   `| fit DBSCAN eps=0.5 min_samples=2 src_ip_numeric`: `src_ip_numeric` 필드를 기준으로 `DBSCAN` 클러스터링을 수행합니다. `eps`는 이웃을 정의하는 반경, `min_samples`는 클러스터를 형성하기 위한 최소 샘플 수입니다.
*   `| where cluster = -1`: `DBSCAN`은 클러스터에 속하지 않는 노이즈 데이터를 `cluster=-1`로 할당합니다. 이들을 필터링하여 이상 징후로 간주할 수 있습니다.

**결과 해석:**
`cluster=-1`로 필터링된 결과는 기존의 일반적인 로그인 실패 패턴에 속하지 않는, 즉 '새로운' 또는 '희귀한' 로그인 시도 패턴을 의미할 수 있습니다. 이는 공격자가 새로운 IP나 계정을 사용하여 시스템에 접근을 시도하는 징후일 수 있습니다.

## MLTK Assistant를 활용한 클러스터링

MLTK 앱의 **Assistant** 기능을 활용하면 클러스터링 모델을 쉽게 구축할 수 있습니다.

1.  MLTK 앱으로 이동하여 **Assistant** 탭을 클릭합니다.
2.  **Clustering** 섹션에서 **Cluster Events**를 선택합니다.
3.  **Data Source**에서 `index=tutorial_logs sourcetype=linux_secure "Failed password"`를 입력합니다.
4.  **Fields to Cluster**에서 `src_ip`, `user` 등 클러스터링할 필드를 선택합니다.
5.  **Algorithm**에서 `K-Means` 또는 `DBSCAN`을 선택하고, 필요한 파라미터(k, eps, min_samples)를 조정합니다.
6.  **Train** 버튼을 클릭하여 모델을 학습시킵니다.
7.  학습이 완료되면 **Explore Clusters** 탭에서 각 클러스터의 특성을 확인하고, **Detect Anomalies** 탭에서 이상 징후로 분류된 이벤트를 확인할 수 있습니다.

## 다음 이야기

이번 파트에서는 클러스터링을 통해 비정형 데이터에서 유사한 패턴을 그룹화하고, 새로운 패턴이나 소수 그룹을 이상 징후로 탐지하는 방법을 실습했습니다. 이는 알려지지 않은 위협이나 내부자 위협을 탐지하는 데 매우 효과적인 기법입니다.

다음 파트에서는 **"[Hands-on] 예측 분석 및 분류 (Predictive Analytics & Classification)"**를 주제로, 과거 데이터를 학습하여 미래 이벤트를 예측하거나, 새로운 이벤트를 자동으로 분류하는 지도 학습(Supervised Learning) 기반의 머신러닝 기법을 알아보겠습니다.

---
