---
layout: post
title: "Webhook Week 3: Go Skeleton (controller-runtime)"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook go senior-series
---

## kubebuilder

```bash
kubebuilder init --domain mycorp.com
kubebuilder create webhook --group apps --version v1 --kind Deployment --defaulting --programmatic-validation
```

## Webhook struct
```go
type DeploymentValidator struct {
    Client client.Client
}

func (v *DeploymentValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    d := obj.(*appsv1.Deployment)
    if d.Spec.Replicas == nil || *d.Spec.Replicas == 0 {
        return nil, errors.New("replicas must be > 0")
    }
    return nil, nil
}
```

## 자가평가
### Q1. 도구? **kubebuilder**. 정답 1.
### Q2. interface? **CustomValidator / CustomDefaulter**. 정답 1.
### Q3. errors? **return error → admission deny**. 정답 1.
### Q4. warnings? **deny 안 함, 경고만**. 정답 1.
### Q5. client? **K8s 자원 조회 시**. 정답 1.
