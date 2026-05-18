---
layout: post
title: "Fleet Week 4: Infrastructure Provider (CAPA, CAPG, CAPZ)"
date: 2026-05-18 23:30:00 +0900
categories: cluster-api fleet platform senior-series
---

## CAPA (AWS)

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata: { name: team-a-aws }
spec:
  region: us-east-1
  controlPlaneLoadBalancer: { scheme: internet-facing }
  networkSpec: ...
```

VPC, subnet, ELB, IAM 자동 생성.

## CAPG (GCP) / CAPZ (Azure)

같은 패턴, cloud별 spec.

## EKS (managed)

CAPA의 EKS variant. 자체 control plane 대신 EKS API 사용.

## Worker Machine

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachineTemplate
spec:
  template:
    spec:
      instanceType: m5.large
      ami: { id: ami-... }
```

## 자가평가
### Q1. CAPA 책임? **AWS infra 자동 생성**. 정답 1.
### Q2. EKS variant? **managed control plane**. 정답 1.
### Q3. Machine Template? **worker 인스턴스 spec**. 정답 1.
### Q4. Multi-cloud? **provider만 다르고 표준 CR**. 정답 1.
### Q5. cluster naming? **CR name 기반 cloud 자원**. 정답 1.

## 다음
Week 5.
