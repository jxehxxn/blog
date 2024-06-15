---
layout: post
title: "[Splunk 12주 완성] 10주차: MLTK를 활용한 이상 징후 탐지 (Anomaly Detection)"
date: 2024-05-03 09:00:00 +0900
categories: splunk security
---

보안 관제 현장에서 가장 골치 아픈 문제는 정적 임계치(Static Threshold)입니다. 초당 로그인 시도가 100번을 넘으면 차단하거나, 트래픽이 평소보다 2배 늘면 경고를 띄우는 방식은 한계가 명확합니다. 공격자가 속도를 늦추어 탐지를 피하거나, 정상적인 업무 시간대에 트래픽이 몰려 오탐이 쏟아지는 경우가 빈번하기 때문입니다.

오늘은 이러한 한계를 극복하기 위해 Splunk의 Machine Learning Toolkit(MLTK)을 심층적으로 살펴보겠습니다. 통계적 기초부터 머신러닝 알고리즘의 실전 적용까지, 3시간 분량의 강의 내용을 이 한 장의 챕터에 담았습니다.

---

### 1. 이상 징후 탐지의 통계적 기초

머신러닝으로 넘어가기 전, 우리는 데이터의 '정상'과 '비정상'을 구분하는 수학적 기준을 이해해야 합니다. 가장 널리 쓰이는 두 가지 방법은 표준편차(Standard Deviation)와 사분위수 범위(IQR)입니다.

#### 1.1 표준편차와 Z-Score (Standard Deviation & Z-Score)

표준편차($\sigma$)는 데이터가 평균($\mu$)으로부터 얼마나 떨어져 있는지를 나타내는 지표입니다. 정규 분포(Gaussian Distribution)를 따르는 데이터에서:
- 데이터의 약 68%는 $1\sigma$ 이내에 존재합니다.
- 데이터의 약 95%는 $2\sigma$ 이내에 존재합니다.
- 데이터의 약 99.7%는 $3\sigma$ 이내에 존재합니다.

**Z-Score**는 특정 데이터 포인트가 평균으로부터 몇 표준편차만큼 떨어져 있는지를 수치화한 것입니다:
$$z = \frac{x - \mu}{\sigma}$$

일반적으로 Z-Score가 3보다 크거나 -3보다 작으면 통계적 이상치로 간주합니다.

**Splunk 실전 SPL:**
```spl
index=network_traffic
| stats avg(bytes) as avg, stdev(bytes) as stdev
| eval z_score = (bytes - avg) / stdev
| where abs(z_score) > 3
```
이 쿼리는 네트워크 트래픽 중 통계적으로 유의미한 수준(99.7% 범위를 벗어남)의 급증이나 급감을 찾아냅니다.

#### 1.2 사분위수 범위 (Interquartile Range, IQR)

표준편차는 극단적인 이상치에 민감하게 반응하여 평균을 왜곡시킬 수 있습니다. 반면 **IQR**은 데이터의 중앙 50%에 집중하므로 훨씬 견고(Robust)합니다.

- **Q1 (1사분위수)**: 하위 25% 지점
- **Q3 (3사분위수)**: 하위 75% 지점
- **IQR**: $Q3 - Q1$

Tukey's Fences 방식에 따르면, 다음 범위를 벗어나는 데이터를 이상치로 정의합니다:
$[Q1 - 1.5 \times IQR, Q3 + 1.5 \times IQR]$

**Splunk 실전 SPL:**
```spl
index=network_traffic
| stats p25(bytes) as q1, p75(bytes) as q3
| eval iqr = q3 - q1
| eval lower_bound = q1 - (1.5 * iqr)
| eval upper_bound = q3 + (1.5 * iqr)
| where bytes < lower_bound OR bytes > upper_bound
```

---

### 2. Splunk MLTK 아키텍처와 워크플로우

MLTK는 복잡한 알고리즘을 직접 구현하지 않고도 보안 전문가가 머신러닝을 활용할 수 있게 해줍니다. 핵심은 `fit`과 `apply` 두 명령어입니다.

#### 2.1 fit 명령어: 모델 학습
`fit`은 과거 데이터를 분석하여 패턴을 학습하고, 그 결과를 모델 파일(`.mlmodel`)로 저장합니다.
- **구문**: `| fit <Algorithm> <TargetField> from <FeatureFields> into <ModelName>`
- **예시**: `| fit DensityFunction login_count into login_baseline`

#### 2.2 apply 명령어: 모델 적용
`apply`는 저장된 모델을 불러와 새로운 데이터에 적용합니다.
- **구문**: `| apply <ModelName>`
- **예시**: `| apply login_baseline | where is_outlier=1`

---

### 3. 실전 랩: DGA(Domain Generation Algorithm) 탐지 (1.5시간)

DGA는 공격자가 C2(Command and Control) 서버와의 통신을 위해 무작위로 도메인 이름을 생성하는 기법입니다. `google.com` 같은 정상 도메인과 `asdf1234qwer.com` 같은 DGA 도메인을 머신러닝으로 구분해 보겠습니다.

#### 1단계: 데이터 생성 및 준비 (Synthetic Data Generation)
실습을 위해 정상 도메인과 DGA 도메인이 섞인 가상 데이터를 생성합니다.

```spl
| makeresults count=2000
| eval type = if(random() % 2 == 0, "benign", "dga")
| eval domain = if(type=="benign", 
    mvindex(split("google.com,amazon.com,facebook.com,apple.com,microsoft.com,netflix.com,github.com,stackoverflow.com,linkedin.com,twitter.com", ","), random() % 10),
    lower(random()) . ".com")
```

#### 2단계: 특징 추출 (Feature Engineering)
머신러닝 모델은 텍스트를 직접 이해하지 못하므로, 도메인의 특징을 숫자로 변환해야 합니다.
1. **길이(Length)**: DGA 도메인은 대개 정상 도메인보다 깁니다.
2. **모음 비율(Vowel Ratio)**: 무작위 문자열은 모음(a, e, i, o, u)의 비율이 낮을 가능성이 큽니다.
3. **자음 비율(Consonant Ratio)**: 반대로 자음의 비율이 높습니다.

```spl
| eval domain_len = len(domain)
| eval vowel_count = len(replace(domain, "[^aeiou]", ""))
| eval vowel_ratio = vowel_count / domain_len
| eval consonant_count = domain_len - vowel_count
| eval consonant_ratio = consonant_count / domain_len
```

#### 3단계: 모델 학습 (Model Training)
여러 개의 의사결정 나무를 사용하는 `RandomForestClassifier` 알고리즘을 사용하여 학습을 진행합니다.

```spl
| fit RandomForestClassifier type from domain_len, vowel_ratio, consonant_ratio into dga_detection_model
```

#### 4단계: 검증 및 탐지 (Validation & Detection)
새로운 데이터를 생성하여 학습된 모델이 얼마나 정확하게 DGA를 찾아내는지 확인합니다.

```spl
| makeresults count=500
| eval type = if(random() % 2 == 0, "benign", "dga")
| eval domain = if(type=="benign", 
    mvindex(split("google.com,amazon.com,facebook.com,apple.com,microsoft.com,netflix.com,github.com,stackoverflow.com,linkedin.com,twitter.com", ","), random() % 10),
    lower(random()) . ".com")
| eval domain_len = len(domain)
| eval vowel_count = len(replace(domain, "[^aeiou]", ""))
| eval vowel_ratio = vowel_count / domain_len
| eval consonant_count = domain_len - vowel_count
| eval consonant_ratio = consonant_count / domain_len
| apply dga_detection_model
| table domain, type, predicted(type)
| eval match = if(type == 'predicted(type)', 1, 0)
| stats sum(match) as correct, count as total
| eval accuracy = (correct / total) * 100
```

---

### 4. 결론 및 모범 사례

머신러닝은 마법이 아닙니다. 하지만 보안 전문가의 직관을 자동화하고 확장하는 가장 강력한 도구입니다.

- **작게 시작하세요**: 복잡한 분류 모델을 만들기 전, `DensityFunction`을 활용한 단일 변수 이상 탐지부터 시작하는 것이 좋습니다.
- **특징 추출이 핵심입니다**: 모델의 성능은 알고리즘 자체보다 어떤 특징(Feature)을 선택하느냐에 따라 결정됩니다.
- **지속적인 재학습**: 공격자의 기법은 계속 변합니다. 모델이 최신 위협을 반영할 수 있도록 정기적으로 재학습시켜야 합니다.

정적 임계치의 한계에서 벗어나, 이제 여러분의 데이터를 직접 학습시켜 지능형 보안 관제 체계를 구축해 보시기 바랍니다.
