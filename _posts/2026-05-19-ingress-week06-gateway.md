---
layout: post
title: "Ingress Week 6: GatewayClass + Gateway"
date: 2026-05-19 03:00:00 +0900
categories: gateway-api senior-series
---

## GatewayClass
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: { name: nginx }
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
```

## Gateway
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: prod-gw }
spec:
  gatewayClassName: nginx
  listeners:
    - { name: https, port: 443, protocol: HTTPS, tls: { certificateRefs: [{name: prod-tls}] } }
```

## 자가평가
### Q1. GatewayClass 의미? **인프라 종류**. 정답 1.
### Q2. Gateway? **포트 + hostname + TLS**. 정답 1.
### Q3. listener? **포트 + 프로토콜**. 정답 1.
### Q4. TLS? **certificateRefs**. 정답 1.
### Q5. namespace? **Gateway는 보통 dedicated ns**. 정답 1.
