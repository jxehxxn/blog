---
layout: post
title: "O11y Week 4: Alertmanager — Routing, Grouping, Silencing, Best Practices"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus alerting platform senior-series
---

## 학습 목표

- Alertmanager 설정 한 줄 한 줄.
- Alert routing tree.
- Grouping/Inhibit/Silence.
- Alert 디자인 best practice (symptom vs cause).

## 1. 비유 — "병원 분류·진료실 분배"

Prometheus가 환자 도착 알람을 보냄. Alertmanager는 "심장과 환자는 심장과로, 골절은 정형외과로" 식 라우팅. 같은 환자가 5번 알람 와도 한 번만 처리(grouping).

## 2. Alert 정의 (PrometheusRule)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata: { name: myapp }
spec:
  groups:
    - name: myapp.alerts
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m])) > 0.01
          for: 10m
          labels:
            severity: page
            team: payments
          annotations:
            summary: "5xx > 1% for 10m on {{ $labels.job }}"
            description: "Check dashboard https://grafana.../myapp"
            runbook: "https://wiki.mycorp.com/runbooks/myapp-5xx"
```

핵심:
- `for: 10m`: 조건 10분 지속 시 fire (jitter 회피).
- `labels`: 라우팅 키 (severity, team).
- `annotations`: 사람용 설명 + runbook URL.

## 3. Alertmanager Routing

```yaml
route:
  receiver: default
  group_by: [alertname, cluster, service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - receiver: payments-pager
      matchers:
        - team = payments
        - severity = page
    - receiver: slack-warnings
      matchers:
        - severity = warning
      continue: true
receivers:
  - name: payments-pager
    pagerduty_configs:
      - service_key: <key>
  - name: slack-warnings
    slack_configs:
      - api_url: <webhook>
        channel: "#alerts"
  - name: default
    slack_configs:
      - api_url: <webhook>
        channel: "#monitoring"
```

- `group_by`: 같은 키의 알람을 묶음 (한 번에 발송).
- `group_wait`: 첫 알람 후 N초 대기 (관련 알람 합치기 위해).
- `repeat_interval`: 미해결 알람 다시 보낼 주기.
- `matchers`: 라우팅 조건.
- `continue: true`: 다음 route 도 평가.

## 4. Inhibit — 상위 알람이 하위 알람 silence

```yaml
inhibit_rules:
  - source_matchers: [severity = critical]
    target_matchers: [severity = warning]
    equal: [cluster, service]
```

같은 cluster/service에 critical이 있으면 warning은 무시. "노드 죽음" 알람이 있으면 그 노드의 "CPU 높음" 알람은 silence.

## 5. Silence

UI에서 임시 silence:
- 예정된 maintenance 중.
- 알람이 false positive 인 동안.

```bash
amtool silence add severity=warning alertname=HighCPU --duration=2h
```

## 6. Alert 디자인 — Symptom vs Cause

빅테크 (Google SRE) 표준: **Symptom-based alert**.

비유: 환자가 "두통이 있다"고 알람. "혈관 직경 0.3mm 감소" 알람 X.

- **Symptom**: 사용자 영향 (5xx, latency, 가용성). page.
- **Cause**: 내부 신호 (CPU, queue length). info/warning.

이유: cause 알람이 너무 많아 alert fatigue. symptom으로 집중.

## 7. RED + USE + SLO 기반

빅테크 alert 표준:
- RED: Rate/Errors/Duration 기반 symptom alert.
- USE: Utilization/Saturation/Errors — debug용 dashboard.
- SLO burn rate: error budget 빠르게 소진 시 alert (10주차).

## 8. Runbook 강제

annotation의 runbook URL 누락 시 alert 받은 사람이 막막. 빅테크 표준: alert당 runbook 의무.

## 9. 운영 함정 5선

1. **for 빠뜨림**: 일시 spike도 page.
2. **page vs warning 미구분**: 새벽 알람 폭주.
3. **runbook 없음**: 새 oncall이 막막.
4. **silence 만료 안 함**: 진짜 사고 놓침.
5. **PagerDuty key 노출**: ESO/secret.

## 10. 실습

```bash
# 1. PrometheusRule로 HighErrorRate, HighLatency, PodCrashLooping 3개 작성
# 2. Alertmanager routes (payments-pager, slack-warnings, default)
# 3. inhibit rule 추가
# 4. 의도적으로 5xx 발생 → 알람 흐름 확인
# 5. silence 명령으로 일시 차단
```

## 11. 자가평가 퀴즈

### Q1. `for: 10m`의 가치?
1. **10분 지속해야 fire — jitter false alarm 방지**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q2. group_by 의 효과?
1. **같은 키 알람 묶어 한 번에 발송**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q3. inhibit rule의 가치?
1. **상위 알람이 하위 알람 silence — alert fatigue 감소**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q4. symptom vs cause alert?
1. **symptom 우선 page, cause는 debug용**
2. 반대
3. 같음
4. 무관

**정답: 1.**

### Q5. runbook annotation 의무 이유?
1. **알람 받은 oncall이 즉시 대응할 수 있게**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 5: Thanos / Mimir]에서 long-term + HA Prometheus를 다룹니다.
