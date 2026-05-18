---
layout: post
title: "Mesh Week 11: Linkerd 비교 + Service Mesh 선택 기준"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh linkerd istio platform senior-series
---

## 학습 목표

- Linkerd 아키텍처.
- Istio 대비 강점/약점.
- Cilium Service Mesh 위치.
- 결정 트리.

## 1. Linkerd 한 줄

CNCF graduated. Rust-based "linkerd2-proxy". 단순함·성능 강조.

## 2. Linkerd 아키텍처

- **linkerd-destination**: control plane.
- **linkerd2-proxy** (sidecar, Rust): data plane.
- 자동 mTLS, 자동 RED metric.

설치:
```bash
linkerd install | kubectl apply -f -
linkerd check
```

namespace inject:
```bash
kubectl get deploy myapp -o yaml | linkerd inject - | kubectl apply -f -
```

## 3. Linkerd 강점

1. **가볍다**: linkerd2-proxy Rust → 메모리 5~10MB (Envoy 50~100MB).
2. **단순**: 학습 곡선 짧음.
3. **운영**: defaults sensible, troubleshoot 쉽다.

## 4. Linkerd 약점

1. **기능 적음**: Istio의 fault injection, ServiceEntry, Telemetry CR 등 미지원.
2. **community 작음**: Istio 대비.
3. **L7 정책 약함**: HTTPRoute 일부만.

## 5. Istio vs Linkerd 결정 트리

```
복잡한 기능 (fault injection, multi-cluster, JWT) 필요?
  Yes → Istio
  No  → ↓
  
대규모 cluster (수천 Pod)?
  Yes → Linkerd (가벼움) 우선 고려
  No  → ↓

Vendor 지원·생태계 큰 것 선호?
  Yes → Istio
  No  → Linkerd
```

## 6. Cilium Service Mesh

eBPF 기반. sidecar 없음 (Ambient와 비슷).

장점:
- 자원 효율 (eBPF).
- L4 mesh + CNI 통합.
- 빠름.

단점:
- L7 mesh 기능 제한 (Envoy 별도 띄움).
- Cilium CNI 필수.

빅테크 평가 중. 도입 증가 중.

## 7. 다른 옵션

- **Consul Connect**: HashiCorp.
- **Open Service Mesh**: Microsoft (deprecated).
- **Kuma**: Kong.
- **AWS App Mesh**: AWS 종속 (deprecated 2026).

빅테크 표준: Istio 또는 Linkerd.

## 8. 결정 기준 7개

1. 운영 부담 vs 기능.
2. 기존 Envoy 익숙도.
3. 사내 PKI 통합.
4. Multi-cluster 필요.
5. Ambient mode 의향.
6. 커뮤니티/벤더.
7. 비용 (특히 Cilium 같은 통합 솔루션).

## 9. 빅테크 사례

### Spotify
Istio + 자체 운영 노하우.

### Lululemon
Linkerd. "less is more" 철학.

### Cilium 사용자 증가
일부 빅테크가 Cilium CNI + mesh로 통합.

## 10. 자가평가 퀴즈

### Q1. Linkerd 강점?
1. **가벼움·단순함 (Rust)**
2. 기능 풍부
3. UI
4. 무관

**정답: 1.**

### Q2. Istio 강점?
1. **기능 풍부 + 생태계**
2. 가벼움
3. UI
4. 무관

**정답: 1.**

### Q3. Cilium Service Mesh 특징?
1. **eBPF 기반 + CNI 통합**
2. sidecar 강조
3. UI
4. 무관

**정답: 1.**

### Q4. 선택 시 가장 큰 trade-off?
1. **운영 부담 vs 기능**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. AWS App Mesh의 상태?
1. **deprecated 2026**
2. 표준
3. 강력
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 12: 캡스톤] production mesh.
