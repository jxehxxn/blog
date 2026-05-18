---
layout: post
title: "SRE 보충 1: SLO as Code (Sloth)"
date: 2026-05-19 07:00:00 +0900
categories: sre slo supplement
---

```yaml
version: prometheus/v1
service: payments
slos:
  - name: success
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(http_requests_total{code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total[{{.window}}]))
    alerting:
      page_alert: { labels: { severity: page } }
```

`sloth generate` → PrometheusRule.

## 결론
SLO + alert을 코드로 → GitOps.
