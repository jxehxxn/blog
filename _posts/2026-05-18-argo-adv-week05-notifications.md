---
layout: post
title: "ArgoCD 심화 Week 5: Notifications Customization"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced notifications platform senior-series
---

## 학습 목표

- argocd-notifications 구조.
- Custom trigger / template.
- Multi-channel (Slack/Teams/Email/PagerDuty).

## 1. argocd-notifications-cm

```yaml
data:
  service.slack: |
    token: $slack-token
  service.pagerduty: |
    serviceKeys:
      tier1: $pd-tier1-key
  template.app-sync-failed: |
    message: |
      App {{.app.metadata.name}} sync failed.
      {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
    slack:
      attachments: |
        [{ "title": "{{.app.metadata.name}}", "color": "#ff0000" }]
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
```

## 2. Subscription

Application annotation:
```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: "#payments-alerts"
    notifications.argoproj.io/subscribe.on-sync-failed.pagerduty: tier1
```

## 3. Tier 분리

Tier 0/1/2/3별 다른 channel + escalation.

## 4. Template Variables

`.app`, `.context`, `.recipient`, `.sync` 등.

```go
{{- range .app.status.resources }}
- {{.kind}}/{{.name}}: {{.status}}
{{- end }}
```

## 5. Webhook + SOAR

custom webhook으로 Tines/Demisto 같은 SOAR.

```yaml
service.webhook.tines: |
  url: https://hooks.tines.com/...
  headers:
    - { name: Authorization, value: "Bearer $tines-key" }
```

## 6. Conditional

```yaml
- when: app.status.operationState.phase in ['Failed'] and app.metadata.labels.tier == 'tier0'
  send: [escalation]
```

## 7. 운영 함정

1. 알람 폭주 (Tier 구분 없이).
2. silence 안 되어 같은 사건 반복 알람.
3. webhook key 노출.
4. template syntax 오류.

## 8. 자가평가

### Q1. argocd-notifications 구조?
1. **service + template + trigger + subscription** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Tier 분리?
1. **Tier 별 escalation** 2. 모두 같음 3. UI 4. 무관

**정답: 1.**

### Q3. SOAR 연동?
1. **webhook service** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Conditional?
1. **when 표현식** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. 함정?
1. **알람 폭주** 2. 안전 3. UI 4. 무관

**정답: 1.**

## 9. 다음

[Week 6: Resource hooks].
