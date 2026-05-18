---
layout: post
title: "Cost Week 3: Karpenter"
date: 2026-05-19 01:00:00 +0900
categories: cost karpenter senior-series
---

## Karpenter

AWS 출범, 사실상 표준 K8s autoscaler.

## NodePool / NodeClass
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  template:
    spec:
      requirements:
        - { key: "karpenter.k8s.aws/instance-category", operator: In, values: [c, m, r] }
        - { key: "karpenter.sh/capacity-type", operator: In, values: [spot, on-demand] }
      nodeClassRef:
        name: default
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

## 장점
- 30초 내 노드 추가.
- 정확한 sizing.
- spot 혼합 자동.

## Consolidation
사용량 낮으면 노드 통합. 비용 절감 자동.

## 자가평가
### Q1. Karpenter 출범? **AWS, 표준**. 정답 1.
### Q2. NodePool 가치? **노드 spec 자유**. 정답 1.
### Q3. Consolidation? **사용량 낮음 → 통합**. 정답 1.
### Q4. Scale-up latency? **30초**. 정답 1.
### Q5. Spot 통합? **자동**. 정답 1.
