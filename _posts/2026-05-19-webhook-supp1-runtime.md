---
layout: post
title: "Webhook 보충 1: controller-runtime Webhook"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook supplement
---

## Setup
```go
mgr.GetWebhookServer().Register("/validate-deployment", &webhook.Admission{Handler: &handler{}})
```

## Handler interface
```go
type Handler interface {
    Handle(context.Context, AdmissionRequest) AdmissionResponse
}
```

## 결론
controller-runtime이 boilerplate 감소.
