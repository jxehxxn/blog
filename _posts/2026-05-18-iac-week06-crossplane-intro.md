---
layout: post
title: "IaC Week 6: Crossplane 소개 — 클라우드를 K8s CRD로 다루기"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- Crossplane의 핵심 아이디어 — Kubernetes를 universal control plane으로.
- Provider 설치와 첫 클라우드 자원 생성.
- Managed Resource(MR) 와 Claim 개념을 안다.
- ArgoCD와 결합한 GitOps flow를 본다.

## 1. 비유 — "K8s가 클라우드를 흉내내게 한다"

Terraform: 외부 CLI가 클라우드 API를 push로 호출.
Crossplane: **K8s controller가 클라우드 API를 pull + reconcile**. 즉 클라우드 자원이 K8s CR 처럼 다뤄짐.

비유: 학교 행정실이 외부 위탁 업체(클라우드)에게 "이런 강의실 만들어줘" 라는 발주서(CR)를 보내고, 행정실이 계속 확인하며 발주서대로 잘 됐는지 chk하는 것.

## 2. 설치

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  -n crossplane-system --create-namespace
```

## 3. Provider 설치 (AWS 예시)

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.6.0
```

provider 설치 후 자격증명 제공:

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata: { name: default }
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
```

또는 IRSA로 정적 키 없이 인증.

## 4. 첫 S3 버킷 만들기

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-crossplane-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f bucket.yaml
kubectl get bucket
```

K8s가 클라우드에 실제 S3 버킷 생성. 30초 후 ready.

## 5. Managed Resource (MR)

각 클라우드 자원이 K8s CRD로. AWS provider만 해도 1000+ MR.

- `Bucket` (S3)
- `Instance` (EC2)
- `Cluster` (EKS)
- `Database` (RDS)
- ...

각 spec 필드가 Terraform resource attribute와 거의 1:1.

## 6. Claim과 Composition (다음 주차 미리)

MR을 직접 쓰면 Terraform과 큰 차이가 없습니다. Crossplane의 진짜 가치는 **Composition + XRD**:

- XRD: "MyDatabase" 같은 사내 추상화 CRD 정의.
- Composition: XRD → MR 여러 개로 매핑.
- Claim: 앱팀이 `MyDatabase` 한 줄로 요청 → composition이 RDS + IAM + monitoring + backup 자동 생성.

7주차에서 깊게.

## 7. GitOps + Crossplane

ArgoCD가 Bucket/Composition/XRD YAML을 sync. **인프라가 K8s 리소스니까 ArgoCD가 자연스럽게 관리.**

```yaml
# ArgoCD Application
spec:
  source:
    repoURL: https://github.com/mycorp/infra.git
    path: aws/buckets
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
```

PR → merge → ArgoCD sync → Crossplane이 클라우드 자원 생성. 풀스택 GitOps.

## 8. Terraform 대비 강점

1. **Continuous reconcile**: 누군가 콘솔로 자원 변경 → Crossplane이 자동 복구.
2. **K8s native**: 같은 RBAC, audit, events로 통합.
3. **GitOps 친화**: ArgoCD/Flux 그대로.
4. **추상화**: Composition으로 사내 표준 패키지화 강력.

## 9. Terraform 대비 약점

1. **K8s 의존**: K8s 알아야 함.
2. **생태계 작음**: provider 수, 커뮤니티가 Terraform 대비 적음.
3. **State 보관**: K8s etcd에 있어 etcd 백업 필수.
4. **학습 곡선**: XRD + Composition은 추상화가 추가됨.

## 10. 빅테크 사례

### Upbound (Crossplane 모기업)
사내 모든 인프라가 Crossplane CR. ArgoCD가 인프라 GitOps.

### Adobe
"Adobe Tenant" XRD: 신규 팀 onboarding이 CR 한 개로 끝남. RDS + IAM + Vault + monitoring + alert 자동.

### Grafana Labs
Terraform과 병행. 새로 만드는 인프라는 Crossplane, 기존은 Terraform 유지.

## 11. 실습

```bash
# 1. kind 클러스터 + Crossplane 설치
# 2. AWS provider 설치 + ProviderConfig
# 3. Bucket 만들기 → AWS 콘솔에서 확인
# 4. kubectl delete bucket → AWS 콘솔에서 실제 삭제 확인
# 5. AWS 콘솔에서 bucket attribute 변경 → 30초 후 Crossplane이 복구하는지 확인
```

## 12. 자가평가 퀴즈

### Q1. Crossplane의 핵심 아이디어?
1. **K8s를 universal control plane으로 — 클라우드 자원을 K8s CR로**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q2. MR(Managed Resource)이란?
1. **각 클라우드 자원에 대응되는 K8s CRD 인스턴스**
2. 일반 Pod
3. Helm chart
4. 무관

**정답: 1.**

### Q3. Continuous reconcile의 효과?
1. **콘솔에서 변경해도 자동 복구 → drift 차단**
2. 더 빠름
3. UI
4. 무관

**정답: 1.**

### Q4. Crossplane이 ArgoCD와 잘 맞는 이유?
1. **인프라가 K8s 객체라 ArgoCD가 그대로 sync 가능**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Crossplane의 State 보관 위치?
1. S3
2. **K8s etcd → etcd 백업 필수**
3. Git
4. 무관

**정답: 2.**

## 13. 다음 주차

[Week 7: Composition + XRD]에서 사내 표준 인프라 패키지화를 다룹니다.
