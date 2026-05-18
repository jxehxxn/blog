---
layout: post
title: "O11y 최종 자가평가 — 30문항 종합 (정답+해설)"
date: 2026-05-18 18:00:00 +0900
categories: observability platform senior-series assessment
---

12주 학습자 종합 평가.

## 섹션 A — 기초/Metrics (Q1~Q10)

### Q1. Observability vs Monitoring?
1. **Observability는 unknown unknowns 처리** 2. 같음 3. 빠름 4. 무관

**정답: 1.**

### Q2. 3 대 신호?
1. **Metrics/Logs/Traces** 2. CPU/Mem/Disk 3. UI 4. 무관

**정답: 1.**

### Q3. Cardinality 폭발 원인?
1. **고유값 label** 2. metric 수 3. UI 4. 무관

**정답: 1.**

### Q4. SLI/SLO 차이?
1. **SLI 실측, SLO 목표** 2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q5. Counter 처리?
1. **rate/increase** 2. 직접 3. UI 4. 무관

**정답: 1.**

### Q6. ServiceMonitor 가치?
1. **declarative scrape target** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q7. rate vs increase?
1. **rate 초당, increase 총량** 2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q8. histogram_quantile 필수?
1. **by (le)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q9. RED method?
1. **Rate/Errors/Duration** 2. UI 3. 무관 4. CPU/Mem/Disk

**정답: 1.**

### Q10. Recording rule 가치?
1. **자주 쓰는 query 사전 계산** 2. UI 3. 비용 4. 무관

**정답: 1.**

## 섹션 B — Alerting/Storage/Logs/Traces (Q11~Q20)

### Q11. `for: 10m` 가치?
1. **jitter false alarm 방지** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q12. inhibit rule 가치?
1. **상위 알람이 하위 silence** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q13. symptom vs cause alert?
1. **symptom 우선 page** 2. 반대 3. 같음 4. 무관

**정답: 1.**

### Q14. Thanos sidecar 역할?
1. **TSDB → S3 업로드** 2. UI 3. scrape 4. 무관

**정답: 1.**

### Q15. Multi-cluster Thanos?
1. **각 cluster sidecar → 공통 S3 → 중앙 query** 2. federation 3. UI 4. 무관

**정답: 1.**

### Q16. Loki vs ELK?
1. **Loki label index, ELK full-text** 2. 같음 3. UI 4. 무관

**정답: 1.**

### Q17. LogQL `|=` 의미?
1. **포함 필터** 2. 미포함 3. regex 4. 무관

**정답: 1.**

### Q18. Trace vs Span?
1. **Trace end-to-end, Span 단위** 2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q19. Context propagation 표준?
1. **W3C Trace Context** 2. UI 3. 무관 4. OTel only

**정답: 1.**

### Q20. Tail sampling 가치?
1. **흥미로운 trace만 보관** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## 섹션 C — OTel/Grafana/SLO/Incident (Q21~Q30)

### Q21. OTel 3 컴포넌트?
1. **API/SDK/Collector** 2. UI 3. 무관 4. trace/metric/log

**정답: 1.**

### Q22. Vendor-neutral 의의?
1. **vendor 변경 비용 0** 2. UI 3. 비용 절감 4. 무관

**정답: 1.**

### Q23. Agent vs Gateway 분리?
1. **노드 수집 + 중앙 가공/집계** 2. 같음 3. UI 4. 무관

**정답: 1.**

### Q24. Grafana provisioning?
1. **dashboard as code → drift 차단** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q25. Variables 가치?
1. **N 환경 view** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q26. Error budget 의미?
1. **허용 가능 실패 양** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q27. Multi-burn-rate alerting?
1. **fast/slow burn 둘 다 감지** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q28. ICS의 IC 역할?
1. **단일 의사결정 책임** 2. 기록 3. 외부 소통 4. SME

**정답: 1.**

### Q29. Mitigation 90% 1번?
1. **Rollback** 2. Manual fix 3. Scale up 4. 무관

**정답: 1.**

### Q30. Blameless 본질?
1. **사고 학습 + 은닉 방지** 2. UI 3. 비용 4. 무관

**정답: 1.**

## 채점

- 28~30: Senior 수준.
- 24~27: 미드, SLO/incident 보강.
- 18~23: 주니어, OTel/Thanos 보강.
- ~17: 핵심 재복습.

다음 코스: Istio Service Mesh.
