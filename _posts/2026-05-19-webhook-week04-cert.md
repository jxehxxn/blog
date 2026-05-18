---
layout: post
title: "Webhook Week 4: Cert 관리"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook cert senior-series
---

## 필수
TLS. K8s API server가 webhook을 HTTPS로.

## cert-manager 통합

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: { name: my-webhook-cert }
spec:
  dnsNames: [my-webhook.my-ns.svc, my-webhook.my-ns.svc.cluster.local]
  issuerRef: { kind: Issuer, name: selfsigned }
  secretName: my-webhook-cert
```

## CABundle injection
- cert-manager의 cainjector가 ValidatingWebhookConfiguration에 caBundle 자동 주입.
- annotation: `cert-manager.io/inject-ca-from: my-ns/my-webhook-cert`.

## 자가평가
### Q1. TLS 필수? **API server → webhook HTTPS**. 정답 1.
### Q2. cert-manager? **자동 발급**. 정답 1.
### Q3. CABundle? **API server가 신뢰**. 정답 1.
### Q4. cainjector? **caBundle 자동 주입**. 정답 1.
### Q5. self-signed? **cert-manager로 가능**. 정답 1.
