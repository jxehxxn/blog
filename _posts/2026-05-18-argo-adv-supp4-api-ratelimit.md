---
layout: post
title: "ArgoCD 심화 보충 4: API Rate Limit / Pagination"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced supplement
---

## Rate Limit

ArgoCD 자체 미내장. 권장:
- ingress (NGINX) rate limit.
- API gateway.

## Pagination

`/api/v1/applications?limit=100&offset=0` 식.

## Selector

```bash
GET /api/v1/applications?selector=team=payments
```

label 기반 필터.

## gRPC 성능

대량 호출은 gRPC > REST.

## Watch

`?watch=true` → long-poll 이벤트 stream.

## 결론

API 운영도 빅테크 책임. 외부 보호 + 효율 query.
