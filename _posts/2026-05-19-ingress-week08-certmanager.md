---
layout: post
title: "Ingress Week 8: cert-manager 기초"
date: 2026-05-19 03:00:00 +0900
categories: cert-manager senior-series
---

## 설치
```bash
helm install cert-manager jetstack/cert-manager --set installCRDs=true
```

## Certificate CR
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  secretName: prod-tls
  issuerRef: { name: letsencrypt-prod, kind: ClusterIssuer }
  dnsNames: [app.mycorp.com]
```

자동 발급 + 갱신.

## annotation method
Ingress에 annotation으로 자동 Certificate 생성:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

## 자가평가
### Q1. cert-manager 책임? **TLS 자동 발급/갱신**. 정답 1.
### Q2. Certificate CR? **명시적 정의**. 정답 1.
### Q3. annotation method? **Ingress에 자동**. 정답 1.
### Q4. secretName? **결과 Secret 이름**. 정답 1.
### Q5. issuer? **CA 종류**. 정답 1.
