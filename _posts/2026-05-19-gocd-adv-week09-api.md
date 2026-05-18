---
layout: post
title: "GoCD 심화 Week 9: REST API 풀스택"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd api senior
---

## 학습 목표
- API 인증.
- 핵심 endpoint.
- pagination + filter.
- 자동화 패턴.

## 1. Auth

### Basic
`-u user:password`.

### Personal Access Token
UI에서 발급 (24h~).
```bash
curl -H "Authorization: Bearer <token>" ...
```

빅테크 표준.

## 2. Accept Header

API 버전:
```
Accept: application/vnd.go.cd+json
Accept: application/vnd.go.cd.v9+json
```

## 3. 핵심 Endpoint

### Pipeline 조회
```bash
GET /api/pipelines/payments-api/history?page_size=10
```

### Trigger
```bash
POST /api/pipelines/payments-api/schedule
{ "materials": { ... } }
```

### Stage rerun
```bash
POST /api/stages/payments-api/<counter>/<stage>/run
```

### Approval
```bash
POST /api/stages/payments-api/<counter>/<stage>/run
```

### Agent
```bash
GET /api/agents
PATCH /api/agents/<uuid>   # update
```

### Config
```bash
GET /api/admin/config.xml
POST /api/admin/config.xml
```

## 4. Pagination

```
GET /api/pipelines/.../history?after=<counter>&page_size=100
```

## 5. Webhook

```bash
POST /api/webhooks/github/notify
```

GitHub push trigger.

## 6. 자동화 예 — Backstage Plugin

```typescript
const status = await fetch(`/api/pipelines/${name}/history?page_size=1`, {
  headers: { 'Authorization': `Bearer ${token}` }
});
```

service catalog에 status 표시.

## 7. Rate Limit

기본 없음. 사내 gateway/proxy로 보강.

## 8. 자가평가
### Q1. Auth 표준? **Personal Access Token**. 정답 1.
### Q2. Accept header? **application/vnd.go.cd+json**. 정답 1.
### Q3. Trigger? **POST /api/pipelines/.../schedule**. 정답 1.
### Q4. pagination? **page_size + after**. 정답 1.
### Q5. Webhook? **/api/webhooks/github/notify**. 정답 1.
