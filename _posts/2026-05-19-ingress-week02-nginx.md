---
layout: post
title: "Ingress Week 2: NGINX Ingress Controller"
date: 2026-05-19 03:00:00 +0900
categories: ingress nginx senior-series
---

## 설치
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx
```

## Annotation 예
- `nginx.ingress.kubernetes.io/rewrite-target`
- `nginx.ingress.kubernetes.io/proxy-body-size`
- `nginx.ingress.kubernetes.io/auth-url`

## 자가평가
### Q1. NGINX Ingress? **가장 광범위 사용**. 정답 1.
### Q2. rewrite-target? **URL rewrite**. 정답 1.
### Q3. body size limit? **annotation**. 정답 1.
### Q4. ext auth? **auth-url**. 정답 1.
### Q5. HPA? **Ingress controller도 HPA**. 정답 1.
