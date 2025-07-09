---
layout: post
title:  "Splunk 고급편 Part 1: Splunk Machine Learning Toolkit (MLTK) 소개 및 설치"
date:   2025-07-08 16:00:00 +0900
categories: jekyll update
---

## 시작하며: 데이터의 바다에서 보물을 찾는 새로운 방법

Splunk 초급편과 중급편을 통해 우리는 방대한 로그 데이터를 수집하고, 정규화하며, SPL(Search Processing Language)을 이용해 특정 패턴이나 임계값을 기반으로 위협을 탐지하는 방법을 배웠습니다. 하지만 세상의 모든 위협과 이상 징후를 사람이 일일이 정의한 규칙(Rule)만으로 탐지하는 것은 불가능에 가깝습니다.

*   **새로운 공격 기법:** 매일 새로운 공격 기법이 등장하며, 기존 룰로는 탐지하기 어려운 '알려지지 않은 위협(Unknown Threats)'이 증가하고 있습니다.
*   **정상 행위의 변화:** 사용자나 시스템의 정상적인 행위 패턴은 끊임없이 변화합니다. 고정된 임계값은 너무 많은 오탐(False Positive)을 발생시키거나, 중요한 이상 징후를 놓치는(False Negative) 결과를 초래할 수 있습니다.
*   **데이터의 복잡성:** 데이터의 양과 종류가 폭발적으로 증가하면서, 사람이 모든 데이터를 분석하고 숨겨진 패턴을 찾아내는 것은 불가능합니다.

이러한 한계를 극복하고, 데이터 속에 숨겨진 '보물'(즉, 이상 징후나 위협)을 찾아내기 위한 강력한 도구가 바로 **머신러닝(Machine Learning)**입니다. Splunk는 이러한 머신러닝 기술을 누구나 쉽게 활용할 수 있도록 **Splunk Machine Learning Toolkit (MLTK)**이라는 앱을 제공합니다.

이번 고급편 시리즈에서는 MLTK를 활용하여 기존의 룰 기반 탐지로는 어려웠던 고도화된 이상 징후를 탐지하고, 데이터에서 새로운 통찰력을 얻는 방법을 심층적으로 다루겠습니다. 첫 번째 파트에서는 MLTK가 무엇인지 소개하고, 설치 및 기본적인 인터페이스를 살펴보겠습니다.

## Splunk Machine Learning Toolkit (MLTK)이란?

MLTK는 Splunk 플랫폼 위에서 머신러닝 모델을 구축하고, 학습시키고, 배포하여 데이터를 분석할 수 있도록 돕는 무료 앱입니다. 복잡한 머신러닝 코드를 직접 작성할 필요 없이, SPL 명령어를 통해 다양한 머신러닝 알고리즘을 적용할 수 있도록 추상화되어 있습니다.

**MLTK의 주요 기능 및 구성 요소:**

1.  **Guided Workflows (Assistant):** 머신러닝에 익숙하지 않은 사용자도 쉽게 모델을 구축할 수 있도록 단계별 가이드를 제공합니다. 'Predictive Analytics', 'Clustering', 'Anomaly Detection' 등 주요 사용 사례별로 워크플로우가 구성되어 있습니다.
2.  **Algorithms:** `predict`, `fit`, `apply` 등 SPL 명령어를 통해 `K-Means`, `DBSCAN`, `Logistic Regression`, `Random Forest` 등 다양한 머신러닝 알고리즘을 사용할 수 있습니다.
3.  **Experiments:** 구축된 머신러닝 모델의 성능을 평가하고, 여러 모델을 비교하며 최적의 모델을 선택할 수 있는 환경을 제공합니다.
4.  **Showcases:** MLTK의 다양한 활용 사례를 보여주는 예제 대시보드와 검색 쿼리를 제공하여, 사용자가 MLTK의 기능을 빠르게 이해하고 적용할 수 있도록 돕습니다.

## [Hands-on] MLTK 설치 및 기본 설정

MLTK는 Splunkbase에서 다운로드하여 Splunk Enterprise에 설치하는 앱입니다. Splunk Enterprise 7.x 버전 이상에서 동작하며, Python for Scientific Computing (PSC) 라이브러리가 필요할 수 있습니다.

**1. MLTK 다운로드**

*   Splunk 웹 화면에서 **Apps > Find More Apps**로 이동합니다.
*   검색창에 "Machine Learning Toolkit"을 입력하고 검색합니다.
*   검색 결과에서 "Splunk Machine Learning Toolkit"을 찾아 **Install** 버튼을 클릭합니다. (Splunkbase 계정 로그인 필요)
*   또는 [Splunkbase 웹사이트](https://splunkbase.splunk.com/app/2890)에서 직접 `.tgz` 파일을 다운로드하여 Splunk 웹 UI의 **Apps > Manage Apps > Install app from file**을 통해 설치할 수도 있습니다.

**2. Python for Scientific Computing (PSC) 설치 (필요시)**

MLTK는 일부 고급 알고리즘을 위해 Python for Scientific Computing (PSC) 라이브러리를 필요로 합니다. MLTK 설치 후, Splunk 웹 UI에 PSC 설치를 권장하는 메시지가 나타날 수 있습니다.

*   **Apps > Manage Apps**로 이동하여 "Splunk Machine Learning Toolkit" 앱을 클릭합니다.
*   'Setup' 탭에서 PSC 설치 관련 안내를 따릅니다. 일반적으로 Splunkbase에서 PSC 앱을 다운로드하여 MLTK와 동일한 방식으로 설치하면 됩니다.

**3. MLTK 앱 인터페이스 둘러보기**

설치가 완료되면 Splunk 웹 화면의 왼쪽 상단 앱 메뉴에서 "Machine Learning Toolkit" 앱을 선택하여 실행합니다.

*   **Home:** MLTK의 주요 기능과 워크플로우를 한눈에 볼 수 있는 대시보드입니다.
*   **Assistant:** 'Predictive Analytics', 'Clustering', 'Anomaly Detection' 등 사용 사례별로 모델 구축 과정을 단계별로 안내합니다. 이 시리즈의 Hands-on 실습에서 주로 활용할 기능입니다.
*   **Experiments:** 구축된 모델들을 관리하고, 성능을 비교하며, 재학습하는 공간입니다.
*   **Algorithms:** MLTK에서 제공하는 모든 머신러닝 알고리즘과 그 사용법에 대한 상세한 설명을 볼 수 있습니다.
*   **Showcases:** MLTK의 다양한 활용 예시를 보여주는 대시보드와 SPL 쿼리들이 있습니다. 처음 MLTK를 접하는 사용자에게 매우 유용합니다.

## 다음 이야기

이번 파트에서는 Splunk MLTK의 개념과 설치 과정을 살펴보았습니다. 이제 여러분의 Splunk는 단순한 검색 엔진을 넘어, 데이터에서 스스로 패턴을 학습하고 이상 징후를 찾아내는 '인공지능 분석가'의 역량을 갖추게 되었습니다.

다음 파트에서는 MLTK의 핵심 기능 중 하나인 **"[Hands-on] 시계열 데이터 이상 탐지 (Time Series Anomaly Detection)"**를 주제로, 서버의 CPU 사용량이나 네트워크 트래픽과 같은 시계열 데이터에서 평소와 다른 패턴을 자동으로 탐지하는 방법을 실습해 보겠습니다.

---
