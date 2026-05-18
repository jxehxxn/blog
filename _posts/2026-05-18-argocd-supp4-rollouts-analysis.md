---
layout: post
title: "ArgoCD 보충 4: Argo Rollouts AnalysisRun — Prometheus, Datadog, Web 분석 깊게"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes rollouts supplement
---

9주차의 AnalysisTemplate을 provider별로 깊게 풉니다.

## 1. AnalysisRun vs AnalysisTemplate

- Template: 재사용 가능한 정의.
- Run: Template 인스턴스 (실제 측정 1회).

Rollout이 Template를 args 채워 Run을 만듦. UI에서 Run 결과 추적.

## 2. Prometheus Provider

```yaml
provider:
  prometheus:
    address: http://prometheus.monitoring:9090
    query: |
      sum(rate(http_requests_total{service="webapp",code!~"5.."}[2m]))
      /
      sum(rate(http_requests_total{service="webapp"}[2m]))
```

- `successCondition`: `result[0] >= 0.95`
- `failureCondition`: `result[0] < 0.90` (선택)
- `failureLimit`: 연속 실패 N회 → abort
- `inconclusiveLimit`: 연속 inconclusive → human review trigger

함정: query에 raw `{service="webapp"}` 하드코딩하지 말고 args로:

```yaml
query: |
  sum(rate(http_requests_total{service="{{args.svc}}",code!~"5.."}[2m]))
  / sum(rate(http_requests_total{service="{{args.svc}}"}[2m]))
```

## 3. Datadog Provider

```yaml
provider:
  datadog:
    interval: 1m
    query: |
      avg:trace.http.request.errors{service:webapp}.as_rate()
```

DD-API-KEY와 DD-APP-KEY를 Secret으로 주입.

빅테크 케이스: Datadog APM은 SLO 자체 객체가 있어 query를 SLO ID로 줄 수 있음.

## 4. Web Provider

임의의 HTTP endpoint를 호출.

```yaml
provider:
  web:
    url: "https://qa.mycorp.com/api/health?service=webapp"
    timeoutSeconds: 10
    jsonPath: "{$.errorRate}"
```

응답 JSON의 `errorRate` 필드를 result로. 사내 QA 시스템·custom metric 연동.

## 5. Job Provider

K8s Job 실행 결과(exit code)로 판단.

```yaml
provider:
  job:
    spec:
      template:
        spec:
          containers:
            - name: smoke-test
              image: mycorp/smoke:latest
              command: ["./run.sh"]
          restartPolicy: Never
```

Exit 0이면 success. 통합 테스트 슈트 직접 돌리는 패턴.

## 6. Wavefront / NewRelic / CloudWatch

각 provider도 동일 구조. 회사 표준 메트릭 시스템에 맞춰 선택.

## 7. 다중 metric 결합

```yaml
metrics:
  - name: success-rate
    successCondition: result[0] >= 0.95
    provider: { prometheus: {...} }
  - name: latency-p95
    successCondition: result[0] <= 200
    provider: { prometheus: {...} }
  - name: business-checkout
    successCondition: result[0] >= 0.98
    provider: { prometheus: {...} }
```

3종 모두 통과해야 promote. 한 종이라도 failureLimit 도달 → abort.

## 8. 운영 패턴

### Pre-rollout vs Post-rollout

```yaml
strategy:
  canary:
    steps:
      - analysis:
          templates: [smoke]
        # rollout 전 사전 검사
      - setWeight: 10
      - analysis:
          templates: [success-rate, latency]
      - pause: { duration: 5m }
      - setWeight: 50
      - analysis:
          templates: [success-rate, latency, business]
      - pause: { duration: 10m }
      - setWeight: 100
      - analysis:
          templates: [success-rate, latency, business]
```

단계마다 깊이 다른 분석.

### Background analysis

```yaml
strategy:
  canary:
    analysis:
      templates: [success-rate]
      startingStep: 2
```

특정 step부터 계속 백그라운드로 측정. 어떤 단계에서든 임계 초과 시 abort.

## 9. AnalysisRun UI

`kubectl argo rollouts get rollout webapp -w`

각 step의 Analysis 결과·measurement count·실패 사유가 표시. 디버깅의 핵심.

## 10. 빅테크 사례 — Intuit의 4-layer analysis

Intuit (KubeCon 발표):
1. Layer 1: Smoke test (pre-rollout).
2. Layer 2: 5xx error rate (Prometheus).
3. Layer 3: Latency p95 (Prometheus).
4. Layer 4: Business metric (Web provider, checkout/payment 성공률).

4층 모두 통과해야 promote. 자동 abort로 사고의 70% 차단.

## 11. 함정 5선

1. Query에 hardcoded service name → 재사용 불가.
2. Interval 너무 짧음 → noise abort.
3. failureLimit 1 → 일시 jitter도 abort.
4. Inconclusive 처리 누락 → run이 영원히 멈춤.
5. Datadog token 노출 위험 → ESO 같은 secret manager로 주입.

## 12. 결론

AnalysisTemplate는 progressive delivery의 핵심. 4종(smoke/error/latency/business) 다층 구성으로 빅테크 수준 무인 promote 운영 가능.
