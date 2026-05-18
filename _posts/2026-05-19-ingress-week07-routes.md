---
layout: post
title: "Ingress Week 7: HTTPRoute / TCPRoute / GRPCRoute"
date: 2026-05-19 03:00:00 +0900
categories: gateway-api senior-series
---

## HTTPRoute
```yaml
spec:
  parentRefs: [{name: prod-gw, namespace: gateway-system}]
  hostnames: [api.mycorp.com]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /v1 } }]
      backendRefs: [{ name: api-v1, port: 80 }]
    - matches: [{ headers: [{ name: x-canary, value: "true" }] }]
      backendRefs: [{ name: api-v2, port: 80 }]
```

## TCPRoute / UDPRoute / TLSRoute
L4 routing.

## GRPCRoute
gRPC service/method 매칭.

## 자가평가
### Q1. HTTPRoute parentRefs? **Gateway 참조**. 정답 1.
### Q2. header match? **canary 등**. 정답 1.
### Q3. TCPRoute? **L4**. 정답 1.
### Q4. GRPCRoute? **service/method 매칭**. 정답 1.
### Q5. cross-ns? **ReferenceGrant 필요**. 정답 1.
