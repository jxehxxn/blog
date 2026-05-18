---
layout: post
title: "IaC Week 9: Crossplane 운영 — Providers, Packages, RBAC, HA"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- Provider 버전 관리와 업그레이드 전략.
- Configuration Package로 사내 XRD/Composition 배포.
- RBAC로 팀별 권한 격리.
- Crossplane core HA + upgrade.

## 1. Provider 운영

### 버전 고정
```yaml
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.6.0   # tag 고정
```

`latest` 절대 금지. provider 변경이 자원 destroy/recreate 일으킬 수 있음.

### Provider 분할
큰 cloud(예: AWS)는 service별 provider 분할 (S3, EC2, RDS 따로). 필요한 것만 설치 → 메모리 절약.

```bash
kubectl get providers
# provider-aws-s3, provider-aws-ec2, provider-aws-rds...
```

### Provider upgrade
```yaml
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.7.0   # new version
```

apply 후 controller 재시작. CRD 변경 사항 확인 (breaking change).

## 2. Configuration Package

사내 XRD + Composition을 OCI 이미지로 패키징해 배포:

```bash
crossplane xpkg build --package-root=. --package-file=mycorp-config.xpkg
crossplane xpkg push xpkg.upbound.io/mycorp/postgresql-config:v1.0.0
```

설치:
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata: { name: mycorp-postgresql }
spec:
  package: xpkg.upbound.io/mycorp/postgresql-config:v1.0.0
```

여러 클러스터에 같은 추상화 배포. ArgoCD가 Configuration CR 자체를 sync.

## 3. RBAC

XR/Claim 권한:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: payments
  name: db-claim
rules:
  - apiGroups: ["db.mycorp.com"]
    resources: ["postgresqlinstances"]
    verbs: ["get", "list", "create", "update"]
```

앱팀은 Claim만, plat팀은 XRD/Composition/Provider까지.

## 4. ProviderConfig 격리

팀별 ProviderConfig 분리해 자격증명 격리:

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata: { name: payments }
spec:
  credentials:
    source: IRSA   # ServiceAccount 기반 IAM
```

ProviderConfig 별 IAM role 다르게 → blast radius 제한.

## 5. HA 배포

```yaml
helm install crossplane crossplane-stable/crossplane \
  -n crossplane-system --create-namespace \
  --set replicas=2 \
  --set leaderElection=true \
  --set resourcesCrossplane.limits.memory=2Gi
```

leader election으로 활성/대기. provider controller도 replicas 가능.

빅테크 표준: replicas 2~3, HPA 설정, PDB로 안전 재시작.

## 6. Backup

Crossplane state = K8s etcd. Velero로 namespace 백업 + cluster manifest backup.

```bash
velero backup create crossplane-backup \
  --include-namespaces crossplane-system \
  --include-cluster-resources=true
```

복원 절차 분기 1회 drill.

## 7. Monitoring

```promql
# managed resource 수
sum(kube_crd_count{group=~".*aws.*"})

# reconcile error rate
sum(rate(controller_runtime_reconcile_errors_total[5m])) by (controller)

# claim 생성 latency
histogram_quantile(0.99, rate(crossplane_claim_reconcile_seconds_bucket[5m]))
```

알람: reconcile error 증가, claim p99 > 30초.

## 8. Upgrade 절차

Crossplane core upgrade는 위험. 절차:
1. CHANGELOG 확인 (breaking change).
2. staging cluster에서 먼저 시도.
3. provider 호환 버전 확인.
4. Helm upgrade.
5. CRD 변경 점검.

## 9. 운영 함정 5선

1. provider `latest` → 예기치 못한 destroy.
2. providerconfig 공유 → blast radius 폭증.
3. HA 미설정 → controller 장애 시 모든 reconcile 멈춤.
4. etcd backup 부재 → 손실 시 자원 재생성 위험.
5. configuration package 버전 미고정 → 클러스터 간 불일치.

## 10. 빅테크 사례

Upbound는 자체 Crossplane control plane을 SaaS로 제공 ("Upbound Cloud"). 멀티 클러스터 단일 UI, 정책, 메트릭. 빅테크가 자체 운영 부담 줄이려고 채택 증가.

## 11. 실습

```bash
# 1. 사내 XRD/Composition을 xpkg로 빌드
# 2. ghcr.io에 push
# 3. 다른 클러스터에 Configuration으로 설치
# 4. RBAC 으로 payments team Claim 권한만 부여
# 5. HA replicas 2 + leader election + 1개 죽여서 무중단 확인
```

## 12. 자가평가 퀴즈

### Q1. Provider 버전 고정의 가치?
1. **예기치 못한 destroy/recreate 차단**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q2. Configuration Package의 가치?
1. **사내 XRD/Composition OCI 패키징 → 다중 클러스터 일관**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q3. ProviderConfig 분리의 가치?
1. **자격증명 격리 → blast radius 제한**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. HA 배포 + leader election?
1. **controller 장애 시 무중단**
2. 더 빠름
3. UI
4. 무관

**정답: 1.**

### Q5. etcd backup의 필요?
1. **Crossplane state가 etcd에 → 손실 시 자원 인지 못함**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 10: Drift detection & remediation]에서 두 도구의 drift 정책을 다룹니다.
