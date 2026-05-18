---
layout: post
title: "GoCD Week 12: 캡스톤 — 풀스택 CD 파이프라인 구축"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd capstone
---

## 캡스톤 개요

12주 학습을 묶어 **MegaCorp의 multi-region CD 파이프라인**을 GoCD로 처음부터 구축.

## 1. 시나리오

- 30 마이크로서비스.
- 3 region (us-east, eu-west, ap-ne).
- dev/stage/prod 3 환경.
- audit + approval 의무 (SOC 2).

목표: 모든 service가 GoCD pipeline으로 multi-region prod까지.

## 2. 산출물

1. **Pipeline as Code**: 모든 30 service yaml.
2. **VSM 다이어그램**: 한 commit → 3 region prod 흐름.
3. **Pipeline Group + RBAC**: 4 팀별 격리.
4. **Elastic Agent (K8s)**: 자동 scaling.
5. **Vault + Slack 통합**: secret 안전, 알람.
6. **Manual Approval**: prod 2 reviewer.
7. **Audit log**: SOC 2 evidence.

## 3. 평가 기준 (Rubric)

총 100점.

### 기준 1: 아키텍처 (40)
- [10] Pipeline 30+ as Code
- [10] Multi-region fan-out
- [10] Pipeline Group + RBAC
- [10] Elastic Agent K8s

### 기준 2: 운영 (30)
- [10] Vault 통합 (secret 평문 0)
- [10] Slack notification (failure 알람)
- [10] Approval + 2 reviewer

### 기준 3: 거버넌스 (30)
- [10] VSM 다이어그램 + 문서
- [10] SOC 2 audit evidence
- [10] DR plan (server backup)

## 4. 모범답안 3종

### A — 스타트업 (5 service)
Pipeline 5 + Docker elastic + Slack. 점수 ~60.

### B — 스케일업 (30 service)
모든 산출물 충족. 점수 ~85.

### C — 엔터프라이즈 (200+ service)
+ multi-server + cross-region DR + 자체 plugin. 점수 ~95.

## 5. 마무리

GoCD를 깊게 다룬 이 시리즈를 마치고, 빅테크 (특히 금융/legacy) CD 운영의 senior 역할 가능. 학습은 첫 production 사고에서 진짜 시작됩니다.

다음 학습 방향:
- 다른 CI/CD 도구 (Course 3: Argo Workflows/Tekton)와 비교.
- 사내 plugin 개발.
- ThoughtWorks community engagement.
