---
layout: post
title: "ArgoCD Week 10: Observability — Metrics, Notifications, Audit"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes observability
---

## 학습 목표

- ArgoCD가 노출하는 Prometheus 메트릭 카테고리를 안다.
- Grafana 대시보드와 알람 룰을 설계한다.
- argocd-notifications로 Slack/Email/PagerDuty 통합한다.
- Audit log를 SIEM(Splunk/OpenSearch)으로 흘려보낸다.

## 1. ArgoCD 메트릭 — 3가지 출처

ArgoCD 각 컴포넌트가 Prometheus 메트릭을 노출:

- **argocd-server (`:8083/metrics`)**: API request, gRPC latency, sessions.
- **argocd-application-controller (`:8082/metrics`)**: Reconcile count/latency, app health, sync status.
- **argocd-repo-server (`:8084/metrics`)**: Git operation duration, render time.

## 2. 핵심 SLI/SLO

```promql
# SLI 1: 평균 reconcile 지연
histogram_quantile(0.95,
  sum(rate(argocd_app_reconcile_bucket[5m])) by (le))

# SLI 2: OutOfSync 비율
count(argocd_app_info{sync_status="OutOfSync"})
/ count(argocd_app_info)

# SLI 3: Sync 실패 비율
sum(rate(argocd_app_sync_total{phase="Failed"}[10m]))
/ sum(rate(argocd_app_sync_total[10m]))

# SLI 4: Repo server git operation latency
histogram_quantile(0.95,
  rate(argocd_git_request_duration_seconds_bucket[5m]))
```

빅테크 SLO 예:
- Reconcile p95 < 30s.
- OutOfSync 비율 < 5% (drift 추적).
- Sync 실패율 < 2%.
- Git request p95 < 5s.

## 3. Grafana 대시보드 (공식)

ArgoCD 본가가 공식 dashboard 제공: https://grafana.com/grafana/dashboards/14584

핵심 패널:
- Applications by health/sync state
- Reconcile latency by destination cluster
- Sync history (시간순)
- Repo server cache hit rate

빅테크 운영: 공식 dashboard를 사내 fork 후 회사 SLO와 매핑.

## 4. Notifications

`argocd-notifications-cm`:

```yaml
data:
  service.slack: |
    token: $slack-token
  template.app-sync-failed: |
    message: |
      App {{.app.metadata.name}} failed to sync at {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
      Status: {{.app.status.operationState.message}}
    slack:
      attachments: |
        [{ "title": "{{.app.metadata.name}}", "color": "#ff0000" }]
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  subscriptions: |
    - recipients:
        - slack:platform-alerts
      triggers:
        - on-sync-failed
```

Application annotation으로 subscription 직접 지정:

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: payments-alerts
```

## 5. PagerDuty 통합

```yaml
service.pagerduty: |
  serviceKeys:
    payments: $pd-key-payments
trigger.on-sync-failed: |
  - when: app.status.operationState.phase in ['Error', 'Failed']
    send: [pagerduty-event]
```

Tier 0 앱만 PagerDuty, 나머지는 Slack only.

## 6. Audit Log

ArgoCD 자체 audit 이벤트 (sync, login, RBAC 변경)는 K8s Event + Application status에 기록. 빅테크는 별도 audit 적재 필요:

- Application controller log → JSON 구조화 → Splunk/OpenSearch.
- API server access log → 마찬가지.
- RBAC 변경 audit → Application status `operationState.startedAt` + `initiatedBy` 로 추적.

### SIEM 적재 예 (Splunk HEC)

```yaml
# argocd-server 로그를 stdout으로
# DaemonSet fluent-bit이 수집 → HEC로 전송
[OUTPUT]
    Name        splunk
    Match       kube.argocd.*
    Host        splunk.mycorp.com
    Port        8088
    Token       ${HEC_TOKEN}
```

11주차 거버넌스에서 audit retention 정책 다룸.

## 7. OpenTelemetry 통합

ArgoCD v2.10+에서 OTel tracing 지원:

```yaml
# argocd-cmd-params-cm
controller.otlp.address: otel-collector:4317
```

분산 추적으로 sync 한 건의 전체 흐름(API → controller → repo-server → cluster apply)을 트레이스. 빅테크 디버깅 강화.

## 8. 운영 함정 5선

1. **메트릭 미수집**: ServiceMonitor 빠뜨림 → Prometheus가 안 봄.
2. **알람 노이즈**: 모든 sync 실패에 페이지 → 새벽 알람 폭주.
3. **OutOfSync 알람 미설정**: drift가 사람 눈에 안 들어옴.
4. **Notification 다중 채널 누락**: Slack down 시 백업 채널 부재.
5. **Audit log 보존 짧음**: SOC 2에서 audit log 1년 요구.

## 9. 빅테크 사례 — Spotify의 ArgoCD SLO

Spotify는 ArgoCD 자체 SLO를 다음과 같이 운영합니다 (발표 기준):
- Reconcile 지연 p95 < 1분.
- Sync 실패율 < 1%.
- Application controller availability 99.9%.

이를 month-over-month 추적해 error budget 관리.

## 10. 실습 과제

1. ServiceMonitor + PrometheusRule로 ArgoCD 메트릭 수집.
2. 공식 Grafana dashboard import.
3. argocd-notifications로 Slack 통합.
4. 사고 시나리오: 일부러 잘못된 manifest push → 알람·notification 흐름 확인.
5. OTel collector 추가 → trace 시각화.

## 11. 자가평가 퀴즈

### Q1. Reconcile 지연 p95 SLO는 보통?
1. 30초 ~ 1분
2. 1초
3. 1시간
4. 무관

**정답: 1.**

### Q2. OutOfSync 비율 알람의 가치는?
1. drift 감지 — 누군가 클러스터를 직접 만짐
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q3. argocd-notifications의 트리거 구조는?
1. when 조건 + send 템플릿 + subscriptions
2. cron
3. cluster scope
4. 무관

**정답: 1.**

### Q4. Audit log 보존이 짧으면?
1. SOC 2/PCI 감사에서 evidence 부족
2. 비용 절감
3. UI 빠름
4. 무관

**정답: 1.**

### Q5. OTel tracing의 가치는?
1. sync 한 건의 전체 흐름을 분산 추적
2. UI 색상
3. 무관
4. 비용

**정답: 1.**

## 12. 다음 주차

[Week 11: Governance, DR]에서는 변경관리·컴플라이언스·재해복구를 GitOps 관점에서 정리합니다.
