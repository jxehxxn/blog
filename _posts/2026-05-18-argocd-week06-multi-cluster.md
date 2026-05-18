---
layout: post
title: "ArgoCD Week 6: 멀티클러스터·멀티테넌트 — Hub-Spoke vs Decentralized"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 학습 목표

- Hub-Spoke와 Decentralized 두 아키텍처의 트레이드오프를 안다.
- ArgoCD에 클러스터를 등록하는 3가지 방식을 구별한다.
- Sharding으로 controller 부하를 분산한다.
- Multi-tenant Project 격리 패턴을 설계한다.

## 1. 두 가지 아키텍처

### (a) Hub-Spoke (중앙 관리)

```
        [ ArgoCD HUB cluster ]
         ↑       ↑       ↑
     ┌───┴─┐ ┌───┴─┐ ┌───┴─┐
     │ k8s │ │ k8s │ │ k8s │   ← spokes
     │ dev │ │stage│ │ prod│
     └─────┘ └─────┘ └─────┘
```

- 1개의 중앙 ArgoCD가 N개 클러스터를 관리.
- 장점: 한곳에서 모두 보임, UI/RBAC 단일.
- 단점: hub 장애가 전체 영향, 네트워크 의존성, hub 자체가 큰 blast radius.

### (b) Decentralized (클러스터마다 ArgoCD)

```
[ argocd ] → k8s-dev
[ argocd ] → k8s-stage
[ argocd ] → k8s-prod
```

- 각 클러스터에 자체 ArgoCD.
- 장점: 격리, blast radius 작음, 자기 클러스터만 관리하니 단순.
- 단점: N개 UI, 통합 메트릭 별도 구축.

### 빅테크 표준: 하이브리드

대부분의 빅테크는 다음을 채택:
- **Production 클러스터에는 각자 ArgoCD** (격리 + DR).
- **Dev/CI 환경은 hub-spoke** (개발자 편의).
- Tier 1과 Tier 2를 분리하는 것이 핵심.

## 2. 클러스터 등록 — 3가지 방식

### (a) `argocd cluster add` (가장 쉬움)

```bash
argocd cluster add my-context --name prod-us-east-1
```

내부적으로 kubeconfig의 ServiceAccount + ClusterRoleBinding을 만들고 ArgoCD에 Secret으로 저장.

### (b) Declarative Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: prod-us-east-1
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: prod-us-east-1
  server: https://k8s-prod-us-east-1.mycorp.com
  config: |
    {
      "bearerToken": "<token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64>"
      }
    }
```

GitOps 자체에 클러스터 등록도 포함 → 메타-gitops.

### (c) IAM Auth / Cloud Provider Plugin

EKS면 IRSA, GKE면 Workload Identity, AKS면 AAD. 토큰을 안 들고도 인증.

## 3. Application Controller Sharding

500+ 앱이면 controller 1개로는 부족. Sharding 옵션:

```yaml
# argocd-cmd-params-cm
controller.sharding.algorithm: round-robin
controller.replicas: "5"
```

각 controller replica가 cluster 그룹을 맡음. 빅테크에서 100+ 클러스터면 sharding이 필수.

## 4. Multi-tenant Project 격리

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
  namespace: argocd
spec:
  description: Payments team
  sourceRepos:
    - 'https://github.com/mycorp/payments-*'
  destinations:
    - namespace: payments
      server: https://k8s-prod-us-east-1.mycorp.com
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ''
      kind: Service
  roles:
    - name: developer
      policies:
        - p, proj:team-payments:developer, applications, sync, team-payments/*, allow
      groups:
        - mycorp:payments-engineers
```

핵심: 한 팀은 자기 sourceRepos·destinations·resources만 만질 수 있음. 7주차 RBAC에서 깊게.

## 5. ApplicationSet으로 cluster fan-out

```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: prod
  template:
    spec:
      destination:
        server: '{{server}}'
      ...
```

`env=prod` 라벨이 붙은 모든 클러스터에 자동 fan-out. 새 prod 클러스터 추가 시 자동 인지.

## 6. 빅테크 운영 함정 5선

1. **hub 단일 장애**: hub-spoke이면 hub multi-AZ HA 필수.
2. **kubeconfig 토큰 만료**: 정기 회전 자동화 필요. IRSA/WI 권장.
3. **Cluster generator의 광역 fan-out**: selector 누락 시 dev 변경이 prod로.
4. **controller sharding 누락**: 1개 controller가 너무 많은 reconcile → 지연.
5. **Project 격리 누락**: 모든 팀이 default project에서 충돌.

## 7. Application Controller 메트릭 모니터링

```promql
sum(rate(argocd_app_reconcile_count[5m])) by (dest_server)
histogram_quantile(0.95, rate(argocd_app_reconcile_bucket[5m]))
argocd_app_info{sync_status="OutOfSync"} 
```

10주차 observability에서 다룸.

## 8. 실습 과제

1. kind로 hub 클러스터 1개 + spoke 클러스터 2개 구성.
2. `argocd cluster add`로 spoke 2개 등록.
3. ApplicationSet `clusters` generator로 spoke 양쪽에 동일 앱 fan-out.
4. 한 spoke만 down시켜 hub UI에서 ConnectionError가 나는 것을 확인.
5. AppProject 만들어 team-A namespace만 허용 → 다른 namespace 시도 시 거부.

## 9. 자가평가 퀴즈

### Q1. Hub-Spoke의 가장 큰 단점은?
1. Hub 장애가 전체 영향
2. UI가 분산
3. 느림
4. 무관

**정답: 1.**

### Q2. Decentralized의 가장 큰 장점은?
1. 격리, blast radius 작음
2. 더 빠름
3. UI 통합
4. 무관

**정답: 1.**

### Q3. Application controller sharding이 필요해지는 시점은?
1. 클러스터·앱 수가 많아져 단일 controller가 reconcile을 못 따라갈 때
2. 시작부터
3. 안 필요
4. UI 느릴 때

**정답: 1.**

### Q4. AppProject의 핵심 가치는?
1. 팀별 source·destination·resource 격리
2. UI 분리
3. 비용 절감
4. 무관

**정답: 1.**

### Q5. Cluster generator selector가 비면?
1. 모든 클러스터에 fan-out — 사고 위험
2. 0개
3. 무작위
4. dev만

**정답: 1.**

## 10. 다음 주차

[Week 7: RBAC, SSO, Project]에서는 SSO(OIDC/SAML) 연결과 RBAC policy 설계의 운영 노하우를 다룹니다.
