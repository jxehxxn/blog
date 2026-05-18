---
layout: post
title: "K8s Deep Week 12: 캡스톤 — 커스텀 Operator + Multi-tenant Cluster 풀스택"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series capstone
---

## 캡스톤 개요

12주간 배운 모든 컴포넌트(컨트롤 플레인·데이터 플레인·네트워크·스토리지·스케줄링·operator·보안·성능·multi-tenancy)를 묶어 **하나의 실전 시스템**을 구축합니다.

## 1. 시나리오

가상 회사 **MegaCorp**의 plaltform 팀에 합류한 첫 분기 과제:

- 200명 엔지니어, 12 팀.
- 300개 마이크로서비스, 클러스터 5개 (region별 dev/stage/prod).
- 사내 DB 자원이 흩어져 있음 → "MyDatabase" CRD로 통합.

CEO 요청: **"3개월 안에 클러스터 격리 + 커스텀 DB operator로 셀프서비스 만들기."**

## 2. 산출물

다음 5종을 모두 제출합니다.

1. **클러스터 아키텍처 다이어그램**: HA 컨트롤 플레인, CNI(Cilium), CSI(EBS), Ingress(NGINX→Gateway API), cert-manager, external-dns 배치.
2. **Custom Operator (Go, kubebuilder)**: MyDatabase CRD + Controller. owner reference, finalizer, status condition 포함.
3. **Multi-tenant 구성 YAML**: 12 팀 namespace + RBAC + Quota + LimitRange + NetworkPolicy + PSA.
4. **성능 운영 문서**: APF 설정, etcd 튜닝, scheduler 튜닝, CoreDNS NodeLocal.
5. **보안 매핑**: PSA restricted, NetworkPolicy default-deny, etcd encryption, audit log SIEM 연동.

## 3. 평가 기준 (Rubric)

총 100점.

### 기준 1: 아키텍처 적정성 (40점)
- [10] HA 컨트롤 플레인 (etcd 3노드, API server 2+)
- [10] CNI 선택과 NetworkPolicy enforcement
- [10] CSI 설정 (StorageClass, snapshot)
- [10] Gateway API + cert-manager + external-dns 통합

### 기준 2: Operator 완성도 (30점)
- [10] CRD 스키마 + status condition
- [10] Owner reference + finalizer + event 사용
- [10] 무한 reconcile 방지 + RBAC marker

### 기준 3: 보안·격리·성능 (30점)
- [10] Multi-tenant 격리 5종(namespace/RBAC/Quota/NP/PSA) 모두
- [10] etcd encryption + audit log
- [10] APF/etcd defrag/CoreDNS 튜닝 등 성능 운영 문서

## 4. 채점 루브릭 + 3종 모범답안

별도 포스트 [K8s Deep Capstone: Rubric과 3종 모범답안] 참고.

## 5. 마무리

이 캡스톤을 통과하면 **빅테크 Senior Platform Engineer 첫 출근에서 즉시 클러스터 설계·운영을 시작할 준비가 된 것**입니다.

다음 코스 후보:
- (다음) Terraform + Crossplane으로 인프라 GitOps 확장.
- 또는 Argo Workflows로 CI 측 GitOps.
- 또는 Observability 풀스택.
