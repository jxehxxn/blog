---
layout: post
title: "O11y Week 12: 캡스톤 — 풀스택 Observability 플랫폼"
date: 2026-05-18 18:00:00 +0900
categories: observability platform senior-series capstone
---

## 캡스톤 개요

12주의 모든 학습을 묶어 **빅테크 수준 풀스택 o11y 플랫폼**을 구축합니다.

## 1. 시나리오

MegaCorp:
- K8s 클러스터 12개, 250 서비스.
- 기존 datadog 사용 중 (비싸짐).
- SLO 운영 시작.
- SRE 문화 정착 필요.

CEO 요청: **"비용 1/3 + SLO 문화 + 인시던트 평균 30분 복구."**

## 2. 산출물

1. **아키텍처 다이어그램**: Prometheus + Thanos + Loki + Tempo + OTel + Grafana + Alertmanager 통합.
2. **OTel collector pipeline**: 3 신호 통합 송출.
3. **SLO + multi-burn-rate alerts**: 표준 service 3 종 SLO.
4. **Dashboards (provisioning)**: RED + USE + SLO 표준.
5. **Incident response plan + game day 결과**: ICS 적용 + postmortem 1건.

## 3. 평가 기준

총 100점.

### 기준 1: 아키텍처 (40)
- [10] 3 신호 모두 HA로
- [10] Thanos/Mimir + S3 long-term
- [10] OTel agent + gateway
- [10] Cardinality 관리 (label 정책)

### 기준 2: 운영 (30)
- [10] SLO + multi-burn-rate alert
- [10] dashboards provisioning + variable
- [10] alertmanager routing + inhibit + silence

### 기준 3: 문화 (30)
- [10] ICS + SEV 표준화
- [10] Blameless postmortem 템플릿 + 실제 작성
- [10] Game day 분기 plan + 실행 결과

## 4. 모범답안

별도 포스트 [O11y Capstone Rubric + 3종 모범답안] 참고.

## 5. 마무리

이 캡스톤 통과 시 빅테크 SRE/Platform 시니어 즉시 시작 가능.
