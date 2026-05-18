---
layout: post
title: "O11y Week 10: SLO / SLI / Error Budget — Google SRE의 운영 철학"
date: 2026-05-18 18:00:00 +0900
categories: observability sre slo platform senior-series
---

## 학습 목표

- SLI/SLO/SLA의 정확한 구분.
- Error budget의 의미와 운영.
- Multi-window multi-burn-rate alerting.
- 빅테크의 SLO 문화.

## 1. 비유 — "약속과 예산"

- SLO = "한 달에 4시간 다운타임까지 OK" 라는 사내 약속.
- Error budget = 그 4시간이 "사용 가능한 예산".
- 예산 다 쓰면 새 feature 배포 중단 + 안정성 작업.
- 예산 남으면 더 과감한 변경 가능.

이게 Google SRE의 핵심.

## 2. SLI 선택

좋은 SLI:
- 사용자 경험을 직접 반영 (latency, availability).
- 잘 측정 가능 (precise).
- aggregable.

대표:
- **Availability**: 성공 요청 / 전체 요청.
- **Latency**: p95/p99 응답 시간.
- **Throughput**: 처리량.
- **Correctness**: 정답률 (ML 등).

## 3. SLO 정의

```yaml
slo:
  service: payments
  sli:
    success_rate: |
      sum(rate(http_requests_total{job="payments",code=~"2..|3.."}[28d]))
      /
      sum(rate(http_requests_total{job="payments"}[28d]))
  target: 0.999   # 99.9%
  window: 28d
```

28일 rolling window가 표준.

## 4. Error Budget

```
budget = 1 - SLO_target
       = 1 - 0.999 = 0.001 (0.1%)
```

월 100만 요청이면:
```
budget_requests = 1,000,000 × 0.001 = 1,000
```

1,000건 실패까지 허용.

burn rate = 현재 소진 속도 / 평균 소진 속도.
- 1.0 = 평균 페이스, 28일에 budget 정확히 소진.
- 2.0 = 14일에 소진.
- 14.4 = 1일에 소진.

## 5. Multi-window Multi-burn-rate Alerting

Google SRE 표준. fast/slow burn 둘 다 감지.

```yaml
- alert: HighErrorBudgetBurn
  expr: |
    (
      job:slo_errors_per_request:ratio_rate1h{job="payments"} > 14.4 * 0.001
      and
      job:slo_errors_per_request:ratio_rate5m{job="payments"} > 14.4 * 0.001
    )
  for: 2m
  labels: { severity: page }

- alert: SlowErrorBudgetBurn
  expr: |
    (
      job:slo_errors_per_request:ratio_rate6h{job="payments"} > 6 * 0.001
      and
      job:slo_errors_per_request:ratio_rate1h{job="payments"} > 6 * 0.001
    )
  for: 15m
  labels: { severity: warning }
```

빠른 burn (2시간 안에 budget 소진 속도) page.
느린 burn (3일) warning.

## 6. Sloth — SLO as Code

```yaml
version: prometheus/v1
service: payments
slos:
  - name: success
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(http_requests_total{job="payments",code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{job="payments"}[{{.window}}]))
    alerting:
      page_alert: { labels: { severity: page } }
      ticket_alert: { labels: { severity: warning } }
```

`sloth generate` → PrometheusRule 자동 생성.

## 7. Budget Policy

빅테크 운영:
- 90% 소진 → 신규 feature freeze.
- 100% 소진 → 안정성 sprint.
- 70% 미만 → 의도적 chaos engineering으로 학습.

## 8. SLO ≠ 100%

100% availability를 약속하면 새 변경 0이어야 함 → 비현실. SLO는 "허용 가능 실패율"을 명시함으로써 변경 속도를 정당화.

## 9. Multi-SLO Composition

여러 SLO 결합:
- payments service availability 99.9%
- payments service latency p99 < 200ms
- 둘 다 충족이 진짜 SLO.

각각 별도 SLO + 별도 budget.

## 10. 빅테크 사례

### Google
SRE 책의 원조. 모든 service에 SLO + error budget + budget 사용 회의.

### Spotify
Sloth + Grafana 통합. 모든 SLO dashboard 자동.

### Adobe
SLO를 product 메트릭으로 OKR.

## 11. 운영 함정 5선

1. SLO 너무 높음 (99.999% 같은) → 항상 violation.
2. SLO 너무 낮음 (99%) → 의미 없음.
3. budget 소진 시 freeze 안 함 → 의미 없음.
4. multi-burn-rate 미사용 → fast burn 놓침.
5. SLI 선택이 사용자 경험 반영 안 함.

## 12. 실습

```bash
# 1. payments mock service의 SLO 정의 (99.9% availability)
# 2. Sloth로 PrometheusRule 자동 생성
# 3. multi-burn-rate alert 적용
# 4. 의도적 5xx 발생 → burn rate 관찰
# 5. budget 시각화 Grafana dashboard
```

## 13. 자가평가 퀴즈

### Q1. SLI/SLO/SLA?
1. **SLI 실측, SLO 목표, SLA 계약**
2. 같음
3. 무관
4. UI

**정답: 1.**

### Q2. Error budget의 의미?
1. **허용 가능 실패 양 — 변경 속도 정당화**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q3. Multi-burn-rate alerting?
1. **fast/slow burn 둘 다 감지 — 빠른 사고 + 느린 누수**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q4. SLO 100%의 문제?
1. **변경 0이어야 → 비현실 + 혁신 정체**
2. 안전
3. UI
4. 무관

**정답: 1.**

### Q5. Budget 소진 시 표준 정책?
1. **신규 feature freeze + 안정성 작업**
2. 무시
3. UI
4. 무관

**정답: 1.**

## 14. 다음 주차

[Week 11: 인시던트 대응 + 포스트모템]에서 사고 라이프사이클을 다룹니다.
