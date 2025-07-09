---
layout: post
title:  "Splunk 고급편 Part 4: [Hands-on] 예측 분석 및 분류"
date:   2025-07-08 17:30:00 +0900
categories: jekyll update
---

## 시작하며: 미래를 예측하고 데이터를 분류하다

지난 파트에서 우리는 클러스터링을 통해 데이터 속의 숨겨진 패턴을 발견하고, 비정상적인 그룹을 이상 징후로 탐지하는 비지도 학습(Unsupervised Learning) 기법을 배웠습니다. 이번 파트에서는 미리 정의된 '정답(Label)'이 있는 데이터를 학습하여 미래를 예측하거나, 새로운 데이터를 특정 범주로 분류하는 **지도 학습(Supervised Learning)** 기법을 다루겠습니다.

**예측 분석(Predictive Analytics)**은 과거 데이터를 기반으로 미래의 특정 값(예: 다음 시간의 CPU 사용량, 다음 달의 매출)을 예측하는 것입니다. 반면 **분류(Classification)**는 주어진 데이터를 미리 정의된 카테고리(예: 정상/악성, 스팸/정상 메일, 침입/비침입) 중 하나로 할당하는 것입니다.

IT 운영 및 보안 분야에서 예측 분석과 분류는 매우 유용하게 활용될 수 있습니다.

*   **장애 예측:** 서버의 특정 지표(메모리 사용량, 디스크 I/O)를 기반으로 서비스 장애 발생 가능성을 미리 예측
*   **보안 이벤트 분류:** 특정 로그 패턴이 실제 공격인지, 아니면 오탐인지 자동으로 분류
*   **사용자 행위 분류:** 사용자의 로그인 패턴을 학습하여, 비정상적인 로그인 시도(예: 계정 탈취)를 자동으로 분류

MLTK의 `fit` 및 `apply` 명령어를 활용하면 이러한 예측 및 분류 모델을 손쉽게 구축하고 활용할 수 있습니다.

## `fit`과 `apply`: 모델 학습과 적용

*   **`fit` 명령어:** 데이터를 학습하여 모델을 생성합니다. `fit <Algorithm> <target_field> from <features>` 형태로 사용하며, `target_field`는 예측하거나 분류할 대상, `features`는 예측에 사용될 입력 필드들입니다.
*   **`apply` 명령어:** `fit` 명령어로 학습된 모델을 새로운 데이터에 적용하여 예측 또는 분류 결과를 생성합니다.

### [Hands-on] 로그인 성공/실패 분류 모델 구축

Splunk 초급편에서 사용했던 `tutorial_logs` 인덱스의 `linux_secure` 로그를 사용하여, 특정 로그인 시도가 성공할지 실패할지 예측하는 간단한 분류 모델을 구축해 보겠습니다. (실제 보안에서는 더 복잡한 시나리오에 적용됩니다.)

**1. 데이터 준비 및 레이블링**

먼저 모델 학습에 사용할 데이터를 준비하고, 각 이벤트에 '성공' 또는 '실패'라는 레이블을 부여합니다.

```spl
index=tutorial_logs sourcetype=linux_secure
| rex "from (?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) port \d+ (?<authentication_method>\w+)"
| rex "for (invalid user )?(?<user>\w+) from"
| eval login_status = if(match(_raw, "Failed password"), "Failed", "Success")
| table _time, src_ip, user, authentication_method, login_status
```

*   `login_status`: `_raw` 로그에 "Failed password"가 포함되어 있으면 `Failed`, 아니면 `Success`로 분류하는 새로운 필드를 생성합니다. 이 필드가 우리가 예측할 '정답(Label)'이 됩니다.

**2. `fit` 명령어를 이용한 모델 학습 (Logistic Regression)**

이제 `src_ip`, `user`, `authentication_method` 필드를 기반으로 `login_status`를 예측하는 `LogisticRegression` 모델을 학습시킵니다.

```spl
index=tutorial_logs sourcetype=linux_secure
| rex "from (?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) port \d+ (?<authentication_method>\w+)"
| rex "for (invalid user )?(?<user>\w+) from"
| eval login_status = if(match(_raw, "Failed password"), "Failed", "Success")
| fields src_ip, user, authentication_method, login_status
| fit LogisticRegression login_status from src_ip, user, authentication_method into login_classifier
```

**SPL 설명:**

*   `| fit LogisticRegression login_status from src_ip, user, authentication_method into login_classifier`: `LogisticRegression` 알고리즘을 사용하여 `login_status`를 예측하는 모델을 학습시킵니다. `src_ip`, `user`, `authentication_method`가 입력 특성(features)으로 사용됩니다. `into login_classifier`는 학습된 모델을 `login_classifier`라는 이름으로 저장합니다.

**3. `apply` 명령어를 이용한 모델 적용 및 예측**

학습된 모델을 사용하여 새로운 로그인 시도 이벤트에 대해 `login_status`를 예측해 보겠습니다.

```spl
index=tutorial_logs sourcetype=linux_secure
| rex "from (?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) port \d+ (?<authentication_method>\w+)"
| rex "for (invalid user )?(?<user>\w+) from"
| apply login_classifier
| table _time, src_ip, user, authentication_method, predicted(login_status)
```

**SPL 설명:**

*   `| apply login_classifier`: 이전에 `login_classifier`라는 이름으로 저장된 모델을 현재 검색 결과에 적용합니다.
*   `predicted(login_status)`: 모델이 예측한 `login_status` 값을 보여줍니다.

**결과 해석:**
검색 결과에서 `predicted(login_status)` 필드를 확인하면, 모델이 각 로그인 시도에 대해 'Failed' 또는 'Success'로 예측한 결과를 볼 수 있습니다. 실제 데이터와 비교하여 모델의 정확도를 평가할 수 있습니다.

## MLTK Assistant를 활용한 예측/분류

MLTK 앱의 **Assistant** 기능을 활용하면 예측 및 분류 모델을 쉽게 구축할 수 있습니다.

1.  MLTK 앱으로 이동하여 **Assistant** 탭을 클릭합니다.
2.  **Predictive Analytics** 섹션에서 **Predict Numeric Fields** (예측) 또는 **Classify Categorical Fields** (분류)를 선택합니다.
3.  **Data Source**에서 `index=tutorial_logs sourcetype=linux_secure`를 입력합니다.
4.  **Target Field**에서 `login_status`를 선택합니다.
5.  **Features**에서 `src_ip`, `user`, `authentication_method`를 선택합니다.
6.  **Algorithm**에서 `LogisticRegression` 등 원하는 알고리즘을 선택합니다.
7.  **Train** 버튼을 클릭하여 모델을 학습시킵니다.
8.  학습이 완료되면 **Apply Model** 탭에서 모델을 적용한 결과를 확인하고, **Evaluate Model** 탭에서 모델의 성능(정확도, 정밀도, 재현율 등)을 평가할 수 있습니다.

## 다음 이야기

이번 파트에서는 지도 학습 기반의 예측 분석 및 분류 모델을 MLTK를 통해 구축하고 활용하는 방법을 배웠습니다. 이는 특정 이벤트의 발생 가능성을 예측하거나, 복잡한 데이터를 자동으로 분류하여 분석 효율을 높이는 데 매우 유용합니다.

이제 우리는 Splunk의 머신러닝 기능을 통해 데이터 속의 숨겨진 패턴을 발견하고, 이상 징후를 탐지하며, 미래를 예측하는 방법을 익혔습니다. 다음 파트에서는 **"사용자 행위 분석 (User Behavior Analytics, UBA) 개념 및 Splunk UBA 연동"**을 주제로, 개별 이벤트가 아닌 사용자 전체의 행위 패턴을 분석하여 내부자 위협이나 계정 탈취와 같은 고도화된 위협을 탐지하는 방법을 알아보겠습니다.

---
