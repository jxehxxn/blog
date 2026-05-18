---
layout: post
title: "Ingress Week 9: ClusterIssuer — Let's Encrypt, Vault, ACME"
date: 2026-05-19 03:00:00 +0900
categories: cert-manager senior-series
---

## Let's Encrypt
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt-prod }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@mycorp.com
    privateKeySecretRef: { name: le-account-key }
    solvers:
      - http01: { ingress: { class: nginx } }
      - dns01:
          route53: { region: us-east-1 }
```

http-01 (단순) or dns-01 (wildcard 가능).

## Vault PKI
```yaml
spec:
  vault:
    server: https://vault.mycorp.com
    path: pki/sign/mycorp
    auth: { kubernetes: { role: cert-manager } }
```

사내 PKI.

## 자가평가
### Q1. Let's Encrypt solver 종류? **http-01, dns-01**. 정답 1.
### Q2. wildcard? **dns-01만**. 정답 1.
### Q3. Vault PKI? **사내 root**. 정답 1.
### Q4. ACME 표준? **자동 발급 protocol**. 정답 1.
### Q5. rate limit? **LE 5/hour/domain**. 정답 1.
