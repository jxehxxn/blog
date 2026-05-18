---
layout: post
title: "Events Week 10: 보안 — Auth, Secret, Network"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## 1. Source Auth

각 source별 secret 필수. webhook은 HMAC, Kafka는 SASL/TLS, S3는 IAM 등.

## 2. EventBus Auth

NATS token 또는 nkey.

## 3. ServiceAccount

Sensor가 K8s 자원 만들 때 RBAC 필요.

```yaml
spec:
  template:
    serviceAccountName: events-sa
```

최소 권한.

## 4. Trigger 외부 호출

http trigger의 token, AWS credentials, Slack token — secret으로.

## 5. Network

- EventSource Service는 내부만 (외부면 ingress + TLS).
- EventBus는 cluster 내부.

## 6. Audit

모든 sensor trigger → K8s event + 로그 적재.

## 7. 함정
- webhook secret 없음 → 외부 누구나.
- SA 권한 과다.
- token 노출 (env).

## 8. 자가평가
### Q1. webhook secret? **HMAC 검증**. 정답 1.
### Q2. NATS auth? **token/nkey**. 정답 1.
### Q3. Sensor RBAC? **최소 권한 SA**. 정답 1.
### Q4. trigger token 보관? **K8s Secret**. 정답 1.
### Q5. 함정? **webhook secret 누락**. 정답 1.

## 다음
[Week 11: 운영/scale].
