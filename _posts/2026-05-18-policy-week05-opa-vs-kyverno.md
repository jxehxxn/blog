---
layout: post
title: "Policy Week 5: OPA vs Kyverno — 의사결정 기준"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno platform senior-series
---

## 학습 목표

- 두 도구의 차이 정량.
- 시나리오별 권장.
- 하이브리드 가능성.

## 1. 정량 비교

| 항목 | OPA Gatekeeper | Kyverno |
|------|----------------|---------|
| 언어 | Rego | YAML |
| 학습 곡선 | 가파름 | 완만 |
| Mutation | 별도 (제한적) | 강력 |
| Generation | X | 강력 |
| Image verify | 외부 | 내장 |
| External data | sync | context API |
| K8s 외 사용 | 가능 (conftest) | K8s 전용 |
| 공식 library | 광범위 | 광범위 |
| 빅테크 채택 | 광범위 | 증가 |

## 2. 시나리오별

### 시나리오 A: K8s만 + 학습 부담 낮춤
**Kyverno 추천**.

### 시나리오 B: Terraform plan / CI 검사 같이
**OPA 추천** (conftest 활용).

### 시나리오 C: Mutation 풍부
**Kyverno**.

### 시나리오 D: 복잡한 logic + 재사용 모듈
**OPA Rego**.

### 시나리오 E: 공급망 verify
**Kyverno verifyImages 또는 OPA + 외부**.

## 3. 하이브리드

같은 cluster에서 둘 다 가능.
- Kyverno: K8s mutation/generation/image verify.
- OPA: 복잡한 admission + CI 단계 (Terraform).

빅테크 일부가 채택.

## 4. 성능

둘 다 admission webhook. 큰 cluster (수천 Pod/s)에선 webhook latency 중요.

벤치마크: 둘 다 합리적. 정책 수 + 복잡도가 더 큰 영향.

## 5. 의사결정 5 질문

1. K8s 외 사용 (Terraform 검사 등)? → OPA 가산점.
2. mutation 필수? → Kyverno.
3. 팀에 Rego 학습 의향? Yes → OPA, No → Kyverno.
4. image verify 필수? → Kyverno 가산점.
5. 단순한 정책 다수? → Kyverno.

## 6. 빅테크 사례

### Google
OPA 광범위 (사내 도구도 OPA).

### CNCF
ConstraintTemplate library 표준.

### Adobe
Kyverno 채택 (학습 부담 낮음).

### Shopify
verifyImages 위해 Kyverno.

## 7. Migration

OPA → Kyverno 또는 반대.
- Constraint와 ClusterPolicy 간 자동 변환 도구 없음.
- 1:1 재작성.
- 점진 (정책별).

## 8. 자가평가 퀴즈

### Q1. K8s 외 사용?
1. **OPA (conftest)**
2. Kyverno 3. 같음 4. 무관

**정답: 1.**

### Q2. Mutation 필수?
1. **Kyverno**
2. OPA 3. 같음 4. 무관

**정답: 1.**

### Q3. Rego 학습 부담 없음?
1. **Kyverno**
2. OPA 3. 같음 4. 무관

**정답: 1.**

### Q4. 하이브리드 가능?
1. **둘 다 같은 cluster 운영 가능**
2. 충돌 3. 무관 4. 한 개만

**정답: 1.**

### Q5. 의사결정 핵심 변수?
1. **K8s 외 사용 + mutation 필요 + 학습 부담**
2. UI 3. 비용 4. 무관

**정답: 1.**

## 9. 다음

[Week 6: Rego 깊게].
