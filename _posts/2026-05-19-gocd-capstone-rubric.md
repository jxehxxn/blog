---
layout: post
title: "GoCD Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd capstone
---

## 1. 루브릭 (100)

### 아키텍처 (40)
- [10] Pipeline as Code (yaml)
- [10] Multi-region fan-out
- [10] Pipeline Group + RBAC
- [10] Elastic Agent K8s

### 운영 (30)
- [10] Vault secret
- [10] Slack notification
- [10] Manual approval + reviewer

### 거버넌스 (30)
- [10] VSM 문서
- [10] SOC 2 evidence
- [10] DR plan

## 2. 모범답안 A — 스타트업

- 5 service, single region.
- UI 클릭 + yaml 일부.
- Docker elastic.
- Slack only.
- audit 미준비.

점수: ~55.

## 3. 모범답안 B — 스케일업

- 30 service, 3 region.
- 100% yaml PaC.
- Pipeline Group 4 팀.
- K8s elastic.
- Vault + Slack + PagerDuty.
- prod 2 reviewer.
- VSM dashboard.
- SOC 2 매핑 진행.

점수: ~85.

## 4. 모범답안 C — 엔터프라이즈

- 200+ service.
- Multi GoCD server (HA + DR).
- Cross-region.
- 자체 plugin.
- Backstage 통합 (API).
- 모든 evidence 자동.

점수: ~95.

## 5. 평가 시뮬레이션 (실패)

- UI 클릭만 (PaC 없음).
- Approval 없음.
- Secret 평문.

점수: ~25. 재제출.

## 6. 결론

빅테크 GoCD 운영 senior 평가 표준.
