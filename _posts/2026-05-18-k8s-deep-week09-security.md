---
layout: post
title: "K8s Deep Week 9: 보안 — RBAC, Pod Security, NetworkPolicy, etcd 암호화"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series security
---

## 학습 목표

- RBAC 의 4요소(Subject, Verb, Resource, Scope)를 자유롭게 조합한다.
- Pod Security Admission(PSA)의 3 레벨(privileged/baseline/restricted)을 안다.
- NetworkPolicy로 namespace·라벨 단위 트래픽 격리를 구현한다.
- etcd encryption at rest를 활성화한다.

## 1. 비유 — "캠퍼스 보안 5계층"

대학 보안에 비유합니다.
1. **신분증 발급** = Authentication (cert, token, SA).
2. **출입 권한 카드** = Authorization (RBAC).
3. **가방 검사** = Admission control (PSA, OPA).
4. **건물 간 통로 제한** = NetworkPolicy.
5. **기록물 금고 암호** = etcd encryption.

## 2. RBAC — 권한 카드

### 4 요소
- Subject: User, Group, ServiceAccount
- Verb: get, list, watch, create, update, delete
- Resource: pods, deployments, etc.
- Scope: Namespace or ClusterWide

### Role / ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: payments
  name: deployer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
```

### RoleBinding / ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: payments
  name: dev-team
subjects:
  - kind: Group
    name: mycorp:payments-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### 핵심 베스트 프랙티스
- 최소 권한 (least privilege).
- ServiceAccount 별 분리(특히 CI에서 쓰는 토큰).
- `cluster-admin` 줄이지 말 것 — 사고 시 blast radius 무한.
- `verbs: ["*"]` 금지.

## 3. ServiceAccount의 실체

비유: Pod이 자기 학생증 가지고 K8s API에 로그인하는 격.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-sa
  namespace: payments
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: webapp-sa
```

자동으로 토큰이 `/var/run/secrets/kubernetes.io/serviceaccount/token` 에 mount. K8s v1.24+ projected token, 시간제한 자동 갱신.

### IRSA / Workload Identity
- EKS: IRSA — ServiceAccount에 IAM Role 연결.
- GKE: Workload Identity.
- AKS: AAD Pod Identity / Workload Identity.

토큰 없이 클라우드 권한 사용 — 안전.

## 4. Pod Security Admission (PSA)

기존 PodSecurityPolicy(PSP)는 deprecated, K8s v1.25+ PSA가 표준.

3 레벨:
- **privileged**: 제한 없음.
- **baseline**: 알려진 위험 차단 (hostNetwork, privileged 컨테이너 등).
- **restricted**: 강력 제한 (runAsNonRoot, drop ALL capabilities 등).

namespace label로 적용:

```bash
kubectl label ns payments pod-security.kubernetes.io/enforce=restricted
```

이후 namespace에 privileged Pod 만들면 admission 거부.

## 5. NetworkPolicy — 트래픽 방화벽

비유: 학과별로 "공대 학생은 의대 강의실 출입 금지" 같은 규칙. **단, CNI가 이걸 enforce 해야 함** (Calico/Cilium 권장).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-only
  namespace: payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { name: payments }
        - namespaceSelector:
            matchLabels: { name: gateway }
  egress:
    - to:
        - namespaceSelector:
            matchLabels: { name: payments }
        - namespaceSelector:
            matchLabels: { name: kube-system }
          podSelector:
            matchLabels: { k8s-app: kube-dns }
      ports:
        - { port: 53, protocol: UDP }
```

빅테크 표준:
- default-deny 시작 (namespace 안 모든 통신 차단)
- 필요한 ingress/egress만 화이트리스트
- DNS는 항상 허용 (kube-system kube-dns)

## 6. Secrets 보호

### Base64 ≠ 암호화
`kubectl get secret foo -o yaml` 의 data는 base64 인코딩일 뿐, 누구나 디코딩 가능. **etcd에 평문에 가까운 상태로 저장**.

### etcd Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <32-byte-base64>
      - identity: {}
```

API server 시작 옵션 `--encryption-provider-config=` 추가. 이후 secret은 etcd에 AES-CBC로 암호화 저장.

KMS provider (AWS KMS, GCP KMS, HashiCorp Vault)와 통합 권장 — 키 자체를 외부 KMS에서 관리.

### Secret 대신 ESO + Vault
8주차 ArgoCD 강의에서 다룬 패턴. Secret 자체를 줄이고 Vault dynamic으로.

## 7. Audit Log

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  - level: Metadata
    resources:
      - group: ""
        resources: ["*"]
```

API server 옵션 `--audit-policy-file=` + `--audit-log-path=`. SIEM에 적재.

빅테크 표준: Metadata level 전부, Secret/RBAC 변경은 RequestResponse, 7년 보존.

## 8. Image Security

- Image admission으로 sigstore 서명 검증 (Tier 1 Course 7).
- Distroless / scratch base image 권장.
- Image scan: Trivy, Snyk, Clair.
- Private registry mirror.

## 9. Runtime Security

- **Falco**: eBPF 기반 비정상 시스템 콜 감지.
- **Tetragon (Cilium)**: eBPF 기반 process/network/file 정책.
- 둘 다 컨테이너 escape 시도, 비정상 fork·exec 감지.

## 10. 실습

```bash
# 1. ServiceAccount + Role + RoleBinding 으로 한 namespace에 deployment 권한만 부여
# 2. 다른 namespace에서 시도 → forbidden
# 3. namespace에 PSA restricted 라벨
# 4. privileged Pod 만들기 시도 → 거부
# 5. default-deny NetworkPolicy + 필요한 egress만 화이트리스트
# 6. etcd encryption config 적용 후 secret 만들고 etcd dump
```

## 11. 자가평가 퀴즈

### Q1. RBAC subject 3종?
1. **User, Group, ServiceAccount**
2. Pod, Service, Deployment
3. namespace 만
4. 무관

**정답: 1.**

### Q2. PSA restricted 의 효과?
1. **runAsNonRoot, drop ALL capabilities 등 강제**
2. 무제한
3. UI
4. 무관

**정답: 1.**

### Q3. NetworkPolicy enforcement 주체?
1. API server
2. **CNI plugin (Calico/Cilium)**
3. kubelet
4. scheduler

**정답: 2.**

### Q4. Secret이 etcd에 어떻게 저장되나 (기본)?
1. AES-256
2. **base64 인코딩만 (사실상 평문) — encryption at rest 별도 활성 필요**
3. PGP
4. 무관

**정답: 2.**

### Q5. IRSA / Workload Identity의 장점?
1. **Pod이 정적 IAM 키 없이 클라우드 권한 사용**
2. UI
3. 비용 절감
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 10: Performance & Scale]에서 etcd tuning, API priority and fairness, 대규모 클러스터 운영을 다룹니다.
