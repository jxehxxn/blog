---
layout: post
title: "O11y Week 1: 오리엔테이션 — Monitoring vs Observability"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus loki tempo opentelemetry platform senior-series
---

## 학습 목표

- Monitoring과 Observability의 본질 차이.
- 3대 신호(metrics/logs/traces) 의미.
- "Known unknown"과 "Unknown unknown" 의 차이.
- 12주 로드맵.

## 1. 비유 — "건강검진 vs 진료"

Monitoring = 정기 건강검진. 미리 정한 항목(혈압, 콜레스테롤)만 체크. **"이 값이 임계 넘으면 알람"** 식.

Observability = 진료. 환자가 "어디가 아파요" 하면 의사가 다양한 검사·문진으로 **원인을 추론**. **사전에 정의되지 않은 질문에도 답 가능**.

## 2. 한 줄 구분

- **Monitoring**: known unknowns 처리 (예상 가능한 장애).
- **Observability**: unknown unknowns 처리 (예상치 못한 장애를 사후 분석).

빅테크 시대의 분산 시스템은 unknown unknowns가 폭증 → observability가 필수.

## 3. 3대 신호

### Metrics
숫자 시계열. 메모리 사용량, 요청 수, 지연 시간 등. Prometheus.

### Logs
이벤트 텍스트. 에러 메시지, 사용자 동작. Loki/ELK.

### Traces
요청 1개의 분산 호출 경로. Tempo/Jaeger.

각 신호가 다른 질문에 답:
- Metrics: "지금 무엇이 얼마나?"
- Logs: "그 시점에 무슨 일이?"
- Traces: "이 요청이 어떻게 흘렀고 어디서 느렸나?"

빅테크 표준: 세 신호 통합 운영.

## 4. Pull vs Push

### Prometheus의 Pull
Prometheus가 target에 HTTP GET `/metrics`로 수집. service discovery로 target 자동 발견.

장점: target 죽음 즉시 알 수 있음 (up=0). 중앙 통제.
단점: target이 internet 너머면 어려움.

### Push 모델 (StatsD, OpenTSDB)
앱이 직접 메트릭 push.

장점: short-lived batch job에 좋음.
단점: pushgateway 같은 중간 단계 필요.

K8s 환경은 Pull이 자연스러움 (DNS·label로 target 발견).

## 5. Cardinality — 가장 큰 함정

비유: Prometheus는 (metric_name, label_set) 마다 별도 시계열을 만듭니다. label에 user_id 같은 고유값을 넣으면 사용자 수만큼 시계열 폭발 → 메모리·디스크 폭증 → 클러스터 마비.

학부생 룰: **label은 cardinality 낮은 값만** (env, region, status_code O. user_id, request_id X).

## 6. SLI / SLO / SLA

- **SLI**: Service Level Indicator. "실측 지표" (예: 5xx 비율 = 0.3%).
- **SLO**: Service Level Objective. "목표" (예: 5xx 비율 < 0.1%).
- **SLA**: Service Level Agreement. 고객 계약. SLO 미달 시 환불 등.

10주차 깊게.

## 7. 빅테크 사례

### Google
SRE 책의 모태. SLO 운영 + error budget.

### Spotify
Backstage + Prometheus + Grafana 통합 IDP.

### Uber
M3DB(자체 Prometheus 호환) 운영. trillion-scale metrics.

## 8. 12주 로드맵

```
[Metrics] 2주 Prometheus → 3주 PromQL → 4주 Alertmanager → 5주 Thanos
[Logs]    6주 Loki
[Traces]  7주 Tempo → 8주 OpenTelemetry
[UI/Op]   9주 Grafana → 10주 SLO → 11주 incident → 12주 capstone
```

## 9. 실습 (필기형)

"애플리케이션이 30분간 5xx error 5% 발생" 상황. monitoring과 observability 각 관점에서 어떻게 다루는지 한 문단으로.

## 10. 자가평가 퀴즈

### Q1. Observability의 핵심 가치?
1. **unknown unknowns에 사후 답할 수 있음**
2. 비용 절감
3. UI
4. 무관

**정답: 1.**

### Q2. 3대 신호?
1. **Metrics, Logs, Traces**
2. CPU, Mem, Disk
3. UI
4. 무관

**정답: 1.**

### Q3. Cardinality 폭발 흔한 원인?
1. **label에 user_id 같은 고유값**
2. metric 수가 많음
3. UI
4. 무관

**정답: 1.**

### Q4. SLI / SLO 차이?
1. **SLI 실측, SLO 목표**
2. 같음
3. SLA가 SLI
4. 무관

**정답: 1.**

### Q5. Prometheus pull 모델 장점?
1. **target 죽음 즉시 인지 + 중앙 통제**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 2: Prometheus 기초]에서 설치 + 첫 metric.
