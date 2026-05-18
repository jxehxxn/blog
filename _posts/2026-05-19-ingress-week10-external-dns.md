---
layout: post
title: "Ingress Week 10: external-dns"
date: 2026-05-19 03:00:00 +0900
categories: external-dns senior-series
---

## external-dns
K8s Service/Ingress의 hostname → DNS A record 자동.

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.mycorp.com
```

## providers
- Route53, Cloud DNS, Cloudflare, Azure DNS, ...

## TXT registry
ownership tracking — external-dns가 만든 record만 관리.

## 자가평가
### Q1. external-dns 책임? **hostname → DNS record**. 정답 1.
### Q2. provider 다수? **Route53/Cloud DNS/Cloudflare**. 정답 1.
### Q3. TXT registry? **ownership**. 정답 1.
### Q4. annotation? **hostname 명시**. 정답 1.
### Q5. cert-manager 결합? **TLS + DNS 모두 자동**. 정답 1.
