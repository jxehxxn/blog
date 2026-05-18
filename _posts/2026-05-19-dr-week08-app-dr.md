---
layout: post
title: "DR Week 8: Application DR — GitOps + State"
date: 2026-05-19 00:00:00 +0900
categories: dr gitops senior-series
---

## GitOps의 DR 가치
manifest = Git에. cluster 손실 시 ArgoCD가 모두 재배포. RTO 빠름.

## State 분리
- Stateless app: GitOps만으로 충분.
- Stateful: PV + DB DR 별도.

## Secret DR
Vault HA + DR. K8s Secret만은 부족.

## Order of restoration
1. Cluster bootstrap.
2. Secret (Vault).
3. CRDs.
4. Namespaces + Quotas.
5. Stateful (PVC restore).
6. Stateless app.

## 자가평가
### Q1. GitOps DR 가치? **manifest로 빠른 재배포**. 정답 1.
### Q2. Stateless? **GitOps 만으로**. 정답 1.
### Q3. Stateful? **PV + DB DR 별도**. 정답 1.
### Q4. Secret DR? **Vault HA + DR**. 정답 1.
### Q5. 복원 순서? **bootstrap → secret → CRD → ns → stateful → stateless**. 정답 1.
