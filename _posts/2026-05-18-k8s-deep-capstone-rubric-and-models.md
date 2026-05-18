---
layout: post
title: "K8s Deep Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series capstone
---

본 포스트는 [Week 12 캡스톤]의 평가 기준 + 모범답안 3종을 풉니다.

## 1. 채점 루브릭 (100점)

### 기준 1: 아키텍처 적정성 (40)
- [10] HA 컨트롤 플레인 (etcd 3+, API server 2+, scheduler 2+)
- [10] CNI 선택 + NetworkPolicy enforcement
- [10] CSI + StorageClass + Snapshot 정책
- [10] Gateway API + cert-manager + external-dns

### 기준 2: Operator 완성도 (30)
- [10] CRD 스키마 + status condition + RBAC marker
- [10] Owner reference + finalizer + event
- [10] 무한 reconcile 방지 + 단위 테스트

### 기준 3: 보안 · 격리 · 성능 (30)
- [10] Multi-tenant 5종 (NS/RBAC/Quota/NP/PSA)
- [10] etcd encryption + audit log SIEM
- [10] APF + defrag cron + CoreDNS NodeLocal 운영 문서

## 2. 모범답안 A — 스타트업 (50명, 1 클러스터)

### 아키텍처
- kubeadm 단일 region. etcd 3 노드 control plane = worker 분리.
- CNI: Calico (NetworkPolicy 필요).
- CSI: EBS gp3.
- Ingress: NGINX + cert-manager Let's Encrypt.
- Gateway API 미도입(나중).

### Operator
- Kubebuilder로 MyDatabase CRD.
- 단순 owner reference, finalizer 없음(외부 자원 없음).
- 단위 테스트 일부.

### 보안 / 성능
- Namespace 2개 + Quota.
- PSA baseline.
- etcd encryption 미적용 (작은 규모, secret 적음).
- 성능 운영: 50 노드 이하라 기본값 OK.

### 점수: 약 65/100. 통과지만 보안/성능 영역 보강 필요.

## 3. 모범답안 B — 스케일업 (200명, 5 클러스터)

### 아키텍처
- region별 control plane HA (3 노드 etcd + 2 API + 2 scheduler).
- CNI: Cilium (eBPF, kube-proxy 대체, L7 정책).
- CSI: EBS gp3 + EFS for RWX. Snapshot weekly + Velero.
- Ingress: NGINX → Gateway API 전환 중. cert-manager + Vault PKI.
- external-dns로 Route53 자동.

### Operator
- MyDatabase CRD + Controller. owner ref, finalizer (S3 백업 폴더 정리), event 모두.
- Conditions: Ready / Progressing / Degraded.
- 단위 + e2e 테스트.

### 보안 / 성능
- 12 팀 namespace + RBAC + Quota + LimitRange + NetworkPolicy default-deny + PSA restricted.
- etcd encryption (AES) + audit log → Splunk.
- 성능: APF로 controller priority, scheduler percentageOfNodesToScore 30%, CoreDNS NodeLocal.

### 점수: 약 88/100. 시니어 수준.

## 4. 모범답안 C — 엔터프라이즈 (5,000명, 50+ 클러스터)

### 아키텍처
- Cluster API로 클러스터 자체를 GitOps 관리. 5 region × dev/stage/prod.
- HA 모든 영역, etcd 전용 노드, NVMe.
- CNI: Cilium + Hubble 관측.
- CSI: 멀티 — EBS, EFS, Rook-Ceph (멀티클라우드).
- Gateway API 표준, cert-manager + 사내 Vault PKI.
- external-dns 멀티존.

### Operator
- 사내 platform 위에 도메인 operator 수십 개. MyDatabase 외 Tenant, EventBus 등.
- 모든 operator 표준화: conditions, finalizer, event, metrics 노출.
- 단위 + e2e + chaos test.

### 보안 / 성능
- Multi-tenant Tier 분리 (regulated = hard cluster, normal = namespace, dev = vCluster).
- etcd KMS provider (Vault), audit log SIEM 7년 보존.
- 성능: APF 정밀, scheduler percentageOfNodesToScore 10%, CoreDNS NodeLocal + DNSPolicy 표준화, IPVS or eBPF.
- DR drill 주 1회.

### 점수: 약 96/100. 빅테크 운영팀 수준.

## 5. 평가 시뮬레이션 — 실패 사례

가상 제출:
- 컨트롤 플레인 단일 replica
- Operator에 finalizer 없음
- Quota만 있고 NetworkPolicy/PSA 없음
- 성능 운영 문서 부재

점수:
- 기준 1: HA(0) + CNI(7) + CSI(5) + Gateway(0) = 12/40
- 기준 2: CRD(7) + ownerref(3) + reconcile(5) = 15/30
- 기준 3: 격리(3) + etcd(0) + 성능(0) = 3/30
- 합계: 30/100. 재제출.

피드백: "HA 없는 컨트롤 플레인은 production 불가. 보안 격리 5종이 핵심인데 2종만. 성능 문서 자체 부재."

## 6. 마무리

이 루브릭은 빅테크 Platform Senior 면접/실무 평가의 실제 기준에 가깝습니다. 점수보다 약점 진단이 본질.
