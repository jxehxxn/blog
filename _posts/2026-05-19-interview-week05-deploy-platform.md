---
layout: post
title: "Interview Week 5: Design — Deployment Platform"
date: 2026-05-19 08:00:00 +0900
categories: interview design senior-series
---

## Prompt
"Design a deployment platform for 1000 engineers."

## 답
1. **Req**: GitOps, multi-cluster, canary, rollback, audit.
2. **Components**: ArgoCD + ApplicationSet + Rollouts.
3. **Data**: Manifest repo + secret 분리.
4. **Scale**: controller sharding + repo-server HPA.
5. **Failure**: HA + canary upgrade.
6. **Security**: SSO + RBAC + image admission.
7. **Trade-off**: hub vs decentralized.

## 자가평가
### Q1. 도구? **ArgoCD + Rollouts**. 정답 1.
### Q2. multi-cluster fan-out? **ApplicationSet**. 정답 1.
### Q3. Secret? **분리 (ESO + Vault)**. 정답 1.
### Q4. trade-off? **hub vs decentralized**. 정답 1.
### Q5. canary? **Rollouts AnalysisTemplate**. 정답 1.
