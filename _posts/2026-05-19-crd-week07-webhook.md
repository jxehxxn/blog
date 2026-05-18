---
layout: post
title: "CRD Week 7: Validation + Defaulting Webhook"
date: 2026-05-19 06:00:00 +0900
categories: crd controller webhook senior-series
---

## kubebuilder
```bash
kubebuilder create webhook --group=db --version=v1 --kind=MyDatabase --defaulting --programmatic-validation
```

## Defaulter
```go
func (r *MyDatabase) Default() {
    if r.Spec.Replicas == 0 {
        r.Spec.Replicas = 1
    }
}
```

## Validator
```go
func (r *MyDatabase) ValidateCreate() error {
    if r.Spec.Replicas > 10 {
        return errors.New("max 10 replicas")
    }
    return nil
}
```

## 자가평가
### Q1. Defaulter? **자동 default 값**. 정답 1.
### Q2. Validator? **검증 reject**. 정답 1.
### Q3. interface? **Defaulter / Validator**. 정답 1.
### Q4. cert? **cert-manager**. 정답 1.
### Q5. failurePolicy? **Fail (안전)**. 정답 1.
