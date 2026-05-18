---
layout: post
title: "ArgoCD 보충 1: GitOps 이론 깊게 — Reconciliation loop, CAP, Pull vs Push의 본질"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes supplement
---

1주차에서 "Push vs Pull" 으로 정리한 부분을 운영체제·분산시스템 관점으로 깊게 풀어냅니다.

## 1. Reconciliation Loop — Kubernetes의 본질

Kubernetes 자체가 reconciliation loop 기반 시스템입니다. Deployment controller는 desired ReplicaSet과 실제 ReplicaSet을 지속 비교하며 수렴.

GitOps는 이 reconciliation 모델을 **Git까지 확장**한 것입니다. desired state는 etcd가 아니라 Git에 살고, 새로운 controller(ArgoCD)가 Git ↔ etcd를 reconcile.

```
Git (truth) → ArgoCD → etcd (desired) → controllers → live state
                ↑─────────── continuous reconcile ───────────│
```

## 2. CAP 관점

분산시스템에서 CAP(Consistency, Availability, Partition tolerance) 중 2개 선택. GitOps는?

- **Git is C**: Git history는 강한 consistency.
- **ArgoCD reconcile은 eventually consistent**: Git 변경 → 클러스터 반영까지 N초 지연.

따라서 GitOps는 본질적으로 **eventual consistency** 모델. "지금 즉시 Git을 보고 클러스터 상태를 단언할 수 없다" — 이게 운영자가 가져야 할 mental model.

## 3. Pull vs Push — 보안·네트워크 분리

### Push 모델 자세히

```
[CI runner]
  ├─ has cluster kubeconfig
  ├─ network rule: CI → API server allowed
  └─ runs: kubectl apply
[k8s API server]
```

- 자격증명·네트워크 권한이 외부에 있음.
- CI 침해 = production 권한 인수.

### Pull 모델 자세히

```
[k8s cluster]
  ├─ ArgoCD running inside
  ├─ network rule: ArgoCD → git (outbound)
  └─ no inbound from CI
[Git]
```

- 자격증명 없고, inbound 차단.
- 외부 침해는 git에 직접 commit 해야 하며 이는 PR + reviewer + signed commit으로 차단.

## 4. Eventual consistency의 함정

Sync 지연으로 일어나는 운영 함정:

- 긴급 hot fix: PR 머지 → 30초 대기 → 자동 sync. CI push는 즉시이지만 GitOps는 지연이 정상.
- 빠르게 reconcile 하고 싶으면 webhook 사용(Git push 즉시 sync trigger).

### Webhook

```bash
# GitHub webhook URL
https://argocd.mycorp.com/api/webhook
```

repo settings → webhook → push 시 즉시 reconcile trigger. 빅테크 표준.

## 5. Idempotency

GitOps의 모든 변경은 idempotent. 같은 Git state로 N번 sync해도 결과는 동일. 이게 **drift 자동 복구**의 토대.

대비:
- 명령형(`kubectl scale deployment x --replicas=5`)은 idempotent하지 않은 부분이 많음(추가/제거).
- 선언적(`replicas: 5`)은 항상 idempotent.

## 6. Sync atomicity의 한계

K8s API server는 multi-resource transaction이 없습니다. 100개 리소스 sync 중 50번째 실패 시 앞 49개는 적용된 상태. atomic rollback 불가.

대응:
- Sync wave로 의존성 순서 정의.
- PreSync hook으로 사전 검증.
- 실패 시 SyncFail hook으로 알람·정리.

## 7. ArgoCD vs Flux의 reconciliation 차이

- ArgoCD: 단일 Application controller가 Application CR을 봄.
- Flux: Source controller + Kustomize/Helm controller + Notification controller 분리.

Flux의 디자인이 더 microservice-스럽지만 ArgoCD 단일 controller가 운영자에게 더 직관적.

## 8. 미래 — Crossplane + GitOps

K8s manifest뿐 아니라 cloud 인프라(AWS RDS, GCP Spanner)도 Crossplane CR로 표현 → ArgoCD가 GitOps로 관리. 진정한 "everything as code".

## 9. 실습 미니 과제

1. PR 머지 후 sync까지 시간 측정(webhook 없이).
2. Webhook 설정 후 다시 측정 → 차이 비교.
3. 같은 manifest로 sync 5회 반복 → idempotent 확인.
4. 멀티 리소스 sync 중간에 일부러 실패시켜 atomicity 부재 관찰.

## 10. 결론

GitOps는 "kubectl apply 자동화"가 아니라 **eventually consistent reconciliation loop를 git까지 확장한 분산시스템 패턴**입니다. 이 mental model을 갖고 운영해야 함정에 안 빠집니다.
