---
layout: post
title: "Policy Week 1: 오리엔테이션 — Policy as Code의 본질"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno platform senior-series
---

## 학습 목표

- Manual review의 한계.
- Admission control 흐름.
- Validating vs Mutating admission.
- OPA Gatekeeper vs Kyverno 첫 비교.

## 1. 비유 — "법전과 무인 검문소"

도시에 사람 경찰만 두면 (1) 인력 부족 (2) 일관성 없음 (3) 사후 적발. **무인 검문소(admission webhook)** 와 **법전(policy code)** 으로 모든 입국을 자동 검사.

## 2. K8s Admission Flow

```
kubectl apply
  ↓
API server: Auth → Authz → Admission Plugins → Mutating Webhooks → Validating Webhooks → etcd
```

webhook이 정책 위반 발견 시 거부 → kubectl apply 실패.

## 3. 왜 Policy as Code?

- **일관성**: 모든 cluster·namespace에 같은 정책.
- **GitOps**: 정책 자체가 PR.
- **Audit**: 어떤 위반이 언제 차단됐는지 log.
- **Scale**: 수천 namespace 자동 적용.

## 4. OPA Gatekeeper

- CNCF graduated.
- Rego language.
- ConstraintTemplate + Constraint 구조.
- 광범위 사용.

## 5. Kyverno

- CNCF incubating.
- YAML로 정책 작성 (Rego 학습 부담 없음).
- Validation + Mutation + Generation + ImageVerify.
- 사용 증가 추세.

## 6. 한 줄 차이

- **OPA**: 표현력 강력 (Rego), 학습 가파름.
- **Kyverno**: K8s-native YAML, 학습 완만, mutation 강력.

## 7. 도입 시점

- 공급망 보안 (image 서명).
- Compliance (SOC2, PCI).
- Multi-tenant 표준화.
- Security baseline (PSA 부족 부분).

## 8. 12주 로드맵

```
[기초] 1 → 2 OPA → 3 Gatekeeper → 4 Kyverno → 5 비교
[심화] 6 Rego → 7 Mutation → 8 Image admission
[운영] 9 Audit → 10 External → 11 GitOps → 12 캡스톤
```

## 9. 자가평가 퀴즈

### Q1. Policy as Code 핵심?
1. **일관성 + GitOps + audit + scale**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Admission webhook의 위치?
1. **API server의 검증 후 단계**
2. etcd 후 3. UI 4. 무관

**정답: 1.**

### Q3. OPA의 언어?
1. **Rego**
2. YAML 3. Python 4. JS

**정답: 1.**

### Q4. Kyverno의 강점?
1. **YAML만으로 작성, mutation 강력**
2. Rego 3. UI 4. 무관

**정답: 1.**

### Q5. 도입 시점?
1. **공급망/compliance/multi-tenant**
2. 1인 회사 3. UI 4. 무관

**정답: 1.**

## 10. 다음

[Week 2: OPA 기초 + Rego].
