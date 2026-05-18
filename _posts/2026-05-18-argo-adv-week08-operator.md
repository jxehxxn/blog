---
layout: post
title: "ArgoCD 심화 Week 8: ArgoCD Operator로 Multi-tenant"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced operator platform senior-series
---

## 학습 목표

- ArgoCD Operator 설치.
- ArgoCD CR로 다중 instance.
- 팀별 독립 ArgoCD.

## 1. ArgoCD CR

```yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata: { name: payments-argocd, namespace: payments-argocd }
spec:
  server:
    route: { enabled: true }
    replicas: 2
  controller:
    sharding:
      enabled: true
      replicas: 3
  rbac:
    policy: ...
  dex: ...
  ha:
    enabled: true
```

namespace 단위 ArgoCD 인스턴스.

## 2. Multi-tenant 패턴

- 팀별 ArgoCD = strong isolation.
- 단점: 운영 N배.
- 장점: blast radius, 권한 자유.

## 3. namespaceManagement

```yaml
spec:
  namespaceManagement:
    - name: payments-*
      allowManagedBy: true
```

이 ArgoCD가 어떤 namespace를 관리할지.

## 4. Operator + GitOps

ArgoCD CR 자체도 ArgoCD로 sync (chicken-and-egg). 표준 패턴:
1. Master ArgoCD (1개) — bootstrap.
2. Team ArgoCD (N개) — ArgoCD CR로 master가 관리.

## 5. 운영

- Master ArgoCD HA + backup 강력.
- Team ArgoCD는 보통 자원.
- 모니터링 통합.

## 6. 함정

1. Master 장애 → 모든 team ArgoCD 영향.
2. CR 변경의 영향 추적.
3. namespace 충돌.

## 7. 자가평가

### Q1. ArgoCD Operator?
1. **다중 ArgoCD instance 관리** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Multi-tenant 강점?
1. **strong isolation + blast radius** 2. 운영 단순 3. UI 4. 무관

**정답: 1.**

### Q3. 표준 패턴?
1. **Master ArgoCD가 Team ArgoCD 관리** 2. 무관 3. UI 4. 빠름

**정답: 1.**

### Q4. namespaceManagement?
1. **각 ArgoCD가 관리할 namespace 정의** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. 함정?
1. **Master 장애 → 전체 영향** 2. 안전 3. UI 4. 무관

**정답: 1.**

## 8. 다음

[Week 9: ApplicationSet 패턴].
