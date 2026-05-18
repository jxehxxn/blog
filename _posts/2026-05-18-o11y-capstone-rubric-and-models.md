---
layout: post
title: "O11y Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-18 18:00:00 +0900
categories: observability platform senior-series capstone
---

[Week 12 캡스톤] 채점 + 3종 모범답안.

## 1. 루브릭

### 기준 1: 아키텍처 (40)
- [10] 3 신호 HA
- [10] Thanos/Mimir + S3
- [10] OTel agent + gateway
- [10] cardinality 관리

### 기준 2: 운영 (30)
- [10] SLO + multi-burn-rate
- [10] dashboards provisioning
- [10] alertmanager 라우팅

### 기준 3: 문화 (30)
- [10] ICS + SEV
- [10] Blameless postmortem
- [10] Game day 분기

## 2. 모범답안 A — 스타트업

- Prometheus 단일 + Grafana + Loki + OTel collector 1개.
- SLO 1개 service만.
- Alertmanager → Slack only.
- 인시던트 ad-hoc.
- 비용 ~$200/mo.

점수: ~58. 작은 규모 OK.

## 3. 모범답안 B — 스케일업

- Prometheus HA + Thanos + S3 (1년 retention).
- Loki + Tempo 추가.
- OTel agent + gateway, tail sampling.
- 5 service SLO + multi-burn-rate.
- IC + SEV 표준.
- Game day 분기 1회.

점수: ~85.

## 4. 모범답안 C — 엔터프라이즈

- 100+ cluster, Thanos federated.
- Mimir 도입 검토.
- OTel 모든 서비스 + semantic convention 강제.
- SLO 50+ + error budget OKR.
- ICS + 24/7 oncall + chaos engineering 주 1회.
- Datadog cost 70% 절감.

점수: ~95.

## 5. 실패 사례

- Loki 모든 label 살림 → 비용 폭발.
- SLO 없음.
- Postmortem blameful.

점수: ~25.

## 6. 결론

루브릭은 빅테크 SRE/Platform 시니어 평가의 표준.
