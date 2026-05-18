---
layout: post
title: "ArgoCD 심화 보충 3: ApplicationSet Plugin SDK"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced supplement
---

## SDK 패턴

ArgoCD가 HTTP plugin에 POST 요청:
```json
{
  "applicationSetName": "string",
  "input": {
    "parameters": {}
  }
}
```

응답:
```json
{
  "output": {
    "parameters": [
      { "key": "value" }
    ]
  }
}
```

## Auth

bearer token (configMapRef.token).

## Go example

```go
http.HandleFunc("/api/v1/getparams.execute", handler)
```

## Caching

ArgoCD가 requeueAfterSeconds 따라 polling. plugin 측 caching으로 부담 감소.

## Versioning

API v1만 현재. 변경 시 ArgoCD 호환 점검.

## 결론

Plugin generator로 사내 시스템 어떤 것이든 연결.
