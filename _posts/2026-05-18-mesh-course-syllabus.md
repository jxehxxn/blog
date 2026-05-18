---
layout: post
title: "Istio Service Mesh 깊이 알기 (Senior Series 5/19): 12주 강의 계획서"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio envoy platform senior-series
---

## 강의 개요

학부생~주니어가 **빅테크 Senior Platform Engineer 수준의 Service Mesh** 이해에 도달하기 위한 12주 과정. Istio 깊게 + Linkerd 비교 + Ambient mode (sidecarless) 까지.

전제 비유: **Service mesh는 "마이크로서비스의 우편 시스템 + 보안실"**. 100개 서비스가 서로 직접 우편(HTTP)을 보내는 대신, 모든 우편이 우체국(sidecar/proxy)을 통과합니다. 우체국이 (1) 라우팅 (2) 암호화 (3) 인증 (4) 통계 (5) 재시도/timeout 을 표준화. 앱 코드 변경 0.

## 사전 지식

- Kubernetes 기초 (필수, Course 1)
- HTTP/gRPC 기초
- TLS 개념 (PKI, cert)
- Linux network 기초

## 학습 결과

12주 후:

1. Istio production-ready 배포.
2. Traffic management (canary, A/B, mirror, fault injection).
3. mTLS + AuthorizationPolicy로 zero-trust.
4. 분산 추적 자동 활성.
5. Multi-cluster mesh (primary-remote, multi-primary).
6. Ambient mode 이해 + 도입 평가.

## 주차 계획

| 주 | 주제 | 형식 |
|----|------|------|
| 1 | 오리엔테이션 — 왜 service mesh | 강의 |
| 2 | Istio 아키텍처 — control/data plane, Envoy | 강의 |
| 3 | Sidecar injection + VirtualService/DestinationRule | 실습 |
| 4 | Traffic management 깊게 | 실습 |
| 5 | mTLS + AuthorizationPolicy + JWT | 실습 |
| 6 | Observability (Kiali, Jaeger, Prometheus) | 실습 |
| 7 | Gateway + Ingress + Egress | 실습 |
| 8 | Ambient Mode — sidecar 없는 mesh | 강의+실습 |
| 9 | Multi-cluster mesh | 강의 |
| 10 | 운영 — upgrade, performance, troubleshooting | 강의 |
| 11 | Linkerd 비교 + 선택 기준 | 강의 |
| 12 | 캡스톤 — production mesh 구축 | 프로젝트 |

## 보충

- 보충 1: Envoy 깊게 (xDS, filter chain)
- 보충 2: mTLS + CA 관리 깊게
- 보충 3: Ambient mode (ztunnel + waypoint)
- 보충 4: Mesh performance tuning

## 다음 주차

[Week 1: 오리엔테이션]에서 mesh의 본질과 필요 시점을 다룹니다.
