---
layout: post
title: "Mesh Week 12: 캡스톤 — Production Service Mesh"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series capstone
---

## 캡스톤 개요

빅테크 production-ready mesh 구축.

## 1. 시나리오

MegaCorp:
- K8s 5 cluster (us-east-1, eu-west-1, ap-northeast-1 + dev/prod).
- 250 마이크로서비스.
- SOC 2 + mTLS 의무.
- canary 배포 표준 필요.

## 2. 산출물

1. **아키텍처**: Istio multi-primary (region별), east-west gateway.
2. **보안**: mTLS STRICT + deny-by-default AuthorizationPolicy + JWT.
3. **트래픽**: canary template (VS + DR) + Argo Rollouts 통합.
4. **관측**: Kiali + Jaeger + Prometheus.
5. **운영**: canary upgrade plan + game day 결과.

## 3. 루브릭

### 기준 1: 아키텍처 (40)
- [10] multi-primary or primary-remote 합리적 선택
- [10] east-west gateway HA
- [10] Sidecar resource로 메모리 최적
- [10] cert-manager + Vault PKI 통합

### 기준 2: 보안 (30)
- [10] mTLS STRICT
- [10] deny-by-default AuthZ + allowlist
- [10] JWT validation

### 기준 3: 운영 (30)
- [10] canary upgrade plan + 실행 기록
- [10] Telemetry CR로 sampling 최적
- [10] game day + postmortem

## 4. 모범답안

별도 포스트.

## 5. 마무리

이 캡스톤 통과 시 Senior Mesh 운영 즉시 가능.
