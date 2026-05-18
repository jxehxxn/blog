---
layout: post
title: "Ingress Week 3: Traefik"
date: 2026-05-19 03:00:00 +0900
categories: ingress traefik senior-series
---

## Traefik
CRD 기반 (IngressRoute), 자동 service discovery.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
spec:
  routes:
    - match: Host(`app.mycorp.com`) && PathPrefix(`/api`)
      kind: Rule
      services: [{name: api, port: 80}]
```

## 자가평가
### Q1. CRD? **IngressRoute**. 정답 1.
### Q2. matcher 문법? **Host()/PathPrefix()**. 정답 1.
### Q3. 자동 discovery? **standard 기능**. 정답 1.
### Q4. Gateway API 지원? **있음**. 정답 1.
### Q5. NGINX 대비? **CRD가 더 유연, dashboard 강력**. 정답 1.
