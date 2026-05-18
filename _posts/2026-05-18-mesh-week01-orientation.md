---
layout: post
title: "Mesh Week 1: 오리엔테이션 — 왜 Service Mesh인가"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series
---

## 학습 목표

- 마이크로서비스의 5대 문제.
- Service mesh의 본질 — sidecar pattern.
- Istio/Linkerd/Cilium service mesh 비교.
- 도입이 가치 있는 시점.

## 1. 비유 — "관청 + 도로 관리국"

10개 부서(서비스)가 각자 우편 직원·암호화 직원·통계 직원을 두면 비효율. 한 관청(mesh)이 **모든 부서 사이 우편/암호화/통계/접근 통제를 표준화**.

기술적으로: **각 Pod 옆에 sidecar proxy (Envoy)를 두고, 모든 in/out 트래픽이 sidecar를 통과**.

## 2. 마이크로서비스 5대 문제

1. **암호화**: 100개 서비스 간 mTLS를 각자 구현? 미친 짓.
2. **인증/인가**: service-to-service auth를 코드마다.
3. **재시도/timeout/circuit breaker**: 각 언어/SDK마다 다름.
4. **트래픽 분할 (canary)**: app code에 하드코딩.
5. **관측**: trace_id propagation, latency 측정을 app마다.

Mesh가 이 5가지를 **인프라 레이어로 통합**. 앱 코드 변경 0.

## 3. Sidecar Pattern

```
┌────────────────┐
│  Pod           │
│  ┌─────┬─────┐ │
│  │ App │ SC  │ │   SC = sidecar (Envoy)
│  └─────┴─────┘ │
└────────────────┘

App → SC → 다른 Pod의 SC → 다른 App
```

App은 localhost로 sidecar에 요청. sidecar가 외부 통신·mTLS·관측 모두 처리.

## 4. Istio vs Linkerd vs Cilium

| 항목 | Istio | Linkerd | Cilium |
|------|-------|---------|--------|
| 데이터 plane | Envoy | linkerd2-proxy (Rust) | eBPF (sidecarless) |
| Control plane | istiod | linkerd-destination | Cilium Operator |
| 학습 곡선 | 가파름 | 완만 | 중상 |
| 기능 | 광범위 | 최소 (필수만) | 네트워크+mesh 통합 |
| 성능 | 중 | 우수 (가벼움) | 우수 (eBPF) |
| 빅테크 | 광범위 | 중간 | 증가 |

## 5. 도입 가치 시점

도입 부담 큼 (운영/학습/성능). 가치 있는 시점:
- 서비스 수 20+ 이상.
- mTLS 강제 필요 (compliance).
- canary/A-B/mirror 자주.
- 분산 추적이 핵심.

서비스 10개 미만 + 단순 환경에는 over-engineering.

## 6. Ambient Mode (Istio 1.18+)

Sidecar의 자원 부담을 회피. ztunnel(per-node) + waypoint(per-namespace) 로 trade-off.

8주차 깊게.

## 7. 실제 사례

### Lyft (Envoy 원천)
Envoy를 만들어 OSS. 사내 모든 서비스에.

### Google (Istio 원천)
사내 GFE 영감, Istio OSS.

### Buoyant (Linkerd 원천)
가벼움·운영 단순함 강조.

## 8. 12주 로드맵

```
[기초] 1주 오리엔테이션 → 2주 아키텍처 → 3주 sidecar/VS
[운영] 4주 traffic → 5주 보안 → 6주 관측 → 7주 gateway
[심화] 8주 Ambient → 9주 multi-cluster → 10주 운영 → 11주 비교
[캡스톤] 12주
```

## 9. 자가평가 퀴즈

### Q1. Service mesh 본질?
1. **sidecar proxy로 인프라 레이어에 통신 기능 통합**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q2. 마이크로서비스 5대 문제 중 가장 큰 것?
1. **암호화/인증의 분산 구현**
2. UI
3. 색상
4. 무관

**정답: 1.**

### Q3. Istio 데이터 plane?
1. **Envoy**
2. Rust proxy
3. eBPF
4. 무관

**정답: 1.**

### Q4. 도입 가치 시점?
1. **서비스 20+ 이상 + mTLS 필요 등**
2. 항상
3. 1개부터
4. 무관

**정답: 1.**

### Q5. Ambient mode 가치?
1. **sidecar 자원 부담 회피**
2. UI
3. 비용 절감만
4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 2: Istio 아키텍처]에서 control plane / data plane / Envoy를 깊게.
