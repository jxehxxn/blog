---
layout: post
title: "Observability 풀스택 깊이 알기 (Senior Series 4/19): 12주 강의 계획서"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus loki tempo opentelemetry platform senior-series
---

## 강의 개요

학부생~주니어가 **빅테크 Senior Platform/SRE 수준의 Observability** 이해에 도달하기 위한 12주 과정. Metrics(Prometheus + Thanos), Logs(Loki), Traces(Tempo), 그리고 이 셋을 통합하는 OpenTelemetry까지.

전제 비유: **Observability = 우주선의 계기판**. 비행 중인 우주선(서비스)의 상태를 외부에서 알 수 있어야 합니다. 단일 계기(고도)만 보면 부족 — 속도·연료·기온·궤도가 모두 필요. 그리고 사고 시 블랙박스가 있어 "왜 그랬는지" 사후 분석이 가능해야 합니다.

## 사전 지식

- Kubernetes 기초 (필수, Course 1 추천)
- Linux 셸, HTTP 기초
- Go/Python/JS 중 한 가지 (instrumentation)
- 통계 기본 (p99, histogram 의미)

## 학습 결과

12주 후:

1. Prometheus + Alertmanager로 metrics 풀스택 운영.
2. PromQL을 자유롭게 작성 (recording rule + alerting rule).
3. Thanos/Mimir로 long-term + HA 구축.
4. Loki로 cost-effective 로그 적재.
5. Tempo로 distributed tracing.
6. OpenTelemetry로 단일 instrumentation → 3종 backend.
7. SLO/error budget 운영.
8. 인시던트 대응 + blameless postmortem.

## 주차 계획

| 주 | 주제 | 형식 |
|----|------|------|
| 1 | 오리엔테이션 — Observability vs Monitoring | 강의 |
| 2 | Prometheus 기초 — pull model, PromQL basics | 실습 |
| 3 | PromQL 깊게 — rate, histogram, aggregation | 실습 |
| 4 | Alertmanager + alerting best practices | 실습 |
| 5 | Thanos / Mimir — long-term + HA | 강의+실습 |
| 6 | Loki — log 적재 + LogQL | 실습 |
| 7 | Tempo — distributed tracing | 실습 |
| 8 | OpenTelemetry — 단일 instrumentation | 실습 |
| 9 | Grafana — dashboards + datasources | 실습 |
| 10 | SLO / SLI / Error Budget | 강의 |
| 11 | 인시던트 대응 + postmortem | 시뮬레이션 |
| 12 | 캡스톤 — 풀스택 o11y 플랫폼 | 프로젝트 |

## 보충

- 보충 1: Prometheus HA + federation 깊게
- 보충 2: OTel Collector pipeline 설계
- 보충 3: Logs cardinality + cost
- 보충 4: Tracing sampling + correlation

## 다음 주차

[Week 1: 오리엔테이션]에서 모니터링과 observability의 본질 차이를 다룹니다.
