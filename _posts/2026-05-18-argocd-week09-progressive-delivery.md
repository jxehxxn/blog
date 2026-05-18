---
layout: post
title: "ArgoCD Week 9: Progressive Delivery — Argo Rollouts로 Canary, Blue-Green, Analysis"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes progressive-delivery
---

## 학습 목표

- Argo Rollouts와 표준 Deployment의 차이를 안다.
- Canary와 Blue-Green 전략을 yaml로 작성한다.
- AnalysisTemplate으로 Prometheus 메트릭 기반 자동 promotion/abort를 구현한다.
- Service Mesh(Istio, Linkerd)와 결합한 트래픽 분할을 한다.

## 1. 왜 Progressive Delivery인가

전통적 Deployment의 rolling update는 단순합니다. 점진적으로 새 ReplicaSet을 늘리고 옛 것을 줄입니다. 하지만:
- "에러율이 5%를 넘으면 자동 중단" 같은 분석 기반 promotion 없음.
- "20% 트래픽으로만 카나리" 같은 정밀 분할 없음.
- "Blue-Green 후 즉시 cut-over" 없음.

Argo Rollouts가 이 빈자리를 채웁니다.

## 2. Rollout 기본

`Deployment`를 `Rollout`으로 바꾸면 됩니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: webapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: mycorp/webapp:v2.3.4
```

흐름:
1. v2.3.4 ReplicaSet 1개 생성 (10% trafic).
2. 5분 대기.
3. 25%로 늘림.
4. 분석.
5. ... 100%.

## 3. Blue-Green

```yaml
strategy:
  blueGreen:
    activeService: webapp-active
    previewService: webapp-preview
    autoPromotionEnabled: false
    scaleDownDelaySeconds: 600
```

- preview 서비스에 새 ReplicaSet 노출 (스테이지 검증).
- 사람이 promote → active로 트래픽 cut-over.
- 옛 ReplicaSet은 600초 후 scale-down.

## 4. AnalysisTemplate — 메트릭 기반 자동화

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

Rollout에서 참조:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - analysis:
          templates:
            - templateName: success-rate
          args:
            - name: service-name
              value: webapp
      - pause: { duration: 5m }
      - setWeight: 50
```

성공률 95% 이하가 3회 → 자동 abort + rollback. 사람 개입 없이 사고 차단.

Prometheus 외 Datadog, NewRelic, Web, Job, Wavefront, CloudWatch provider 지원. 자세한 분석 패턴은 [보충 4: AnalysisRun] 포스트.

## 5. Service Mesh 통합 — Istio, Linkerd, SMI

```yaml
strategy:
  canary:
    canaryService: webapp-canary
    stableService: webapp-stable
    trafficRouting:
      istio:
        virtualServices:
          - name: webapp-vs
            routes: [primary]
```

`setWeight: 20`이 그냥 ReplicaSet 수가 아니라 **실제 트래픽의 20%**를 의미하게 됩니다 (Istio가 트래픽 분할).

빅테크 표준: Istio + Argo Rollouts 결합. 정밀한 트래픽 분할.

## 6. 사용자 정의 metric — 비즈니스 지표

성공률뿐 아니라 비즈니스 메트릭으로도 가능. 예: 결제 성공률, 평균 응답 시간, 로그인 실패율.

```promql
sum(rate(checkout_success_total[2m])) / sum(rate(checkout_attempts_total[2m]))
```

이게 분석 메트릭이면 "결제 성공률이 떨어지면 카나리 중단" 가능. 진짜 Progressive Delivery.

## 7. 실제 사례 — Intuit의 자동 abort 통계

Intuit는 발표에서 Argo Rollouts + Prometheus 자동 abort로 production 사고의 ~70%를 카나리 단계에서 자동 차단한다고 했습니다. 사람이 새벽에 깨지 않습니다.

## 8. 운영 함정 5선

1. **Pause duration 부족**: 메트릭 적정 평가 전에 다음 단계 → 의미 없는 promotion.
2. **failureLimit 1**: 일시 jitter도 abort → 잦은 false abort.
3. **메트릭 미설정 단계**: setWeight만 있고 analysis 없음 → 자동화 안 됨.
4. **Service Mesh 미연동**: setWeight가 ReplicaSet 수로만 작동 → 트래픽 분할 부정확.
5. **scaleDownDelaySeconds 너무 짧음**: rollback 시 옛 ReplicaSet이 이미 사라짐.

## 9. 실습 과제

1. 표준 Deployment를 Rollout으로 변환.
2. 4단계 canary (10/25/50/100) 작성.
3. Prometheus mock 메트릭으로 AnalysisTemplate 작성.
4. 일부러 5xx 비율을 올려 자동 abort 동작 확인.
5. Blue-Green 변형으로 같은 앱 운영.

## 10. 자가평가 퀴즈

### Q1. AnalysisTemplate의 핵심 가치는?
1. 메트릭 기반 자동 promotion/abort — 사람 개입 최소화
2. UI
3. 무관
4. 비용 절감

**정답: 1.**

### Q2. setWeight만으로 트래픽 분할이 정확한가?
1. ReplicaSet 수 기반이라 정확한 트래픽 % 아님 — Service Mesh 결합 필요
2. 정확함
3. 무관
4. 의미 없음

**정답: 1.**

### Q3. failureLimit 1의 단점은?
1. 일시 jitter도 abort
2. 안전함
3. 비용 절감
4. 무관

**정답: 1.**

### Q4. Blue-Green의 핵심 트레이드오프는?
1. 2배 리소스, 무중단 cut-over
2. 의미 없음
3. 같은 비용
4. 더 빠름

**정답: 1.**

### Q5. 비즈니스 메트릭(checkout 성공률) 기반 abort의 장점은?
1. 기술 메트릭에 안 잡히는 사고도 차단
2. 무관
3. 비용
4. UI

**정답: 1.**

## 11. 다음 주차

[Week 10: Observability]에서는 ArgoCD 메트릭, notifications, audit 로그를 다룹니다.
