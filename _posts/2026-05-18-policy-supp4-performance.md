---
layout: post
title: "Policy 보충 4: Performance Impact + 운영"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno platform senior-series supplement
---

## 1. Admission Latency

webhook = API server 요청에 latency 추가. 보통 10~100ms.

cluster의 admission 부하:
```promql
apiserver_admission_webhook_admission_duration_seconds
```

p99 > 500ms 경고.

## 2. Webhook timeout

```yaml
webhook:
  timeoutSeconds: 10
```

10초 초과면 admission fail (또는 ignore, fail-closed/open 정책).

## 3. Fail Policy

```yaml
failurePolicy: Fail  # 또는 Ignore
```

- Fail: webhook 장애 시 admission deny → 안전.
- Ignore: webhook 장애 시 통과 → 가용성.

빅테크 선택: critical은 Fail, optional은 Ignore.

## 4. HA Webhook

Gatekeeper/Kyverno 자체 HA + PDB. webhook 자체가 single point가 안 되게.

## 5. 정책 수 영향

각 admission마다 모든 정책 평가. 100개 policy = 100번 평가.

대책:
- match 좁게 (`apiGroups`, `kinds`, `namespaces`).
- 자주 안 쓰는 자원 exclude.

## 6. Audit 부하

Background audit이 모든 자원 스캔. 큰 cluster (수만 자원) 에서 자원·시간 부담.

대책:
- audit interval 조절 (60s → 300s).
- audit replica 분리.

## 7. External Data 성능

호출 latency. timeout + cache 필수.

## 8. 운영 메트릭

```promql
gatekeeper_audit_duration_seconds
kyverno_admission_review_duration_seconds
kyverno_policy_execution_duration_seconds
```

p95 알람.

## 9. 결론

Policy 풀스택은 잘 운영하면 zero-trust + compliance + 자동화. 잘못하면 cluster 마비. 시니어 책임.
