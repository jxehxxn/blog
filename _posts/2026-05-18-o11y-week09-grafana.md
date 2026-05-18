---
layout: post
title: "O11y Week 9: Grafana — Dashboards, Datasources, Provisioning"
date: 2026-05-18 18:00:00 +0900
categories: observability grafana platform senior-series
---

## 학습 목표

- Grafana의 multi-datasource 모델.
- Dashboard JSON과 provisioning.
- Variable / Template.
- Alert (Grafana Alerting).

## 1. 비유 — "통합 계기판"

Prometheus/Loki/Tempo가 각각 데이터를 가지고 있고, Grafana가 단일 UI로 시각화. 우주선 조종실의 계기판.

## 2. 설치

kube-prometheus-stack에 기본 포함.

또는:
```bash
helm install grafana grafana/grafana
```

## 3. Datasource

- Prometheus (metrics)
- Loki (logs)
- Tempo (traces)
- 추가: MySQL, PostgreSQL, CloudWatch, ES, ...

## 4. Dashboard

각 panel:
- 종류: graph, table, stat, gauge, logs, traces, geomap, ...
- query: PromQL/LogQL/TempoQL.
- visualization 옵션.
- legend.

## 5. Variables

```
$cluster, $namespace, $pod
```

dashboard 상단의 dropdown으로 선택.

```promql
sum(rate(http_requests_total{cluster="$cluster",namespace="$namespace"}[5m]))
```

같은 dashboard로 N 환경 view.

## 6. Provisioning

dashboard를 UI로만 만들면 drift. GitOps:

```yaml
# grafana-dashboards-configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-dashboard
  labels:
    grafana_dashboard: "1"
data:
  my-dashboard.json: |
    { ... }
```

kube-prometheus-stack의 sidecar가 이 configmap을 자동 import.

datasource provisioning도 비슷 (`grafana_datasource: "1"` label).

## 7. Folder / Permission

빅테크 패턴: 팀별 folder + 권한.

## 8. Grafana Alerting

v8+ alert 시스템. Prometheus rule 외에도:
- query against datasource → 조건 평가.
- 다중 source 결합 (mixed).
- contact point (Slack, PagerDuty, email).
- silencing, mute timing.

장점: dashboard와 alert 통합.
단점: Prometheus rule이 더 표준적 (alertmanager).

빅테크 패턴: Prometheus rule 우선, Grafana alert는 보조.

## 9. Explore Mode

ad-hoc query. metric → click trace → click log → click metric 의 자유로운 흐름. 디버깅 핵심.

## 10. Annotations

dashboard에 deploy/incident 마킹:
```
{deploy_time=2026-05-18T14:00, deploy_version=v1.2.3}
```

panel에 vertical line으로 표시. "이 spike 가 이 deploy 직후" 시각 확인.

## 11. 운영 함정 5선

1. UI에서 dashboard 수정 → 다음 sync에 사라짐.
2. variable cascade 너무 복잡 → 로딩 느림.
3. PromQL 한 query에 1000+ 시리즈 → panel 멈춤.
4. alert rule scatter (Prometheus + Grafana 혼재) → 운영 혼란.
5. permission 부재 → 누구나 prod alert 변경.

## 12. 빅테크 사례

### Spotify
Backstage 안에 Grafana 임베드. dashboard as code.

### Adobe
Grafana Enterprise + datasource permission 강력.

## 13. 실습

```bash
# 1. ConfigMap으로 dashboard 1개 provisioning
# 2. variable cluster/namespace 추가
# 3. metric + log + trace mix panel
# 4. annotation으로 deploy 마킹
# 5. Explore mode로 임의 디버깅
```

## 14. 자가평가 퀴즈

### Q1. Provisioning의 가치?
1. **dashboard as code → drift 차단**
2. UI 색
3. 비용
4. 무관

**정답: 1.**

### Q2. Variables 의 가치?
1. **같은 dashboard로 N 환경 view**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. Explore mode 가치?
1. **ad-hoc 디버깅 (metric → trace → log)**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. Annotation 가치?
1. **deploy/incident 시각화 — 원인 추정 용이**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q5. Grafana Alert vs Prometheus Rule?
1. **Prometheus rule 우선, Grafana alert는 보조**
2. 반대
3. 같음
4. 무관

**정답: 1.**

## 15. 다음 주차

[Week 10: SLO/SLI/Error Budget]에서 운영 철학을 정리.
