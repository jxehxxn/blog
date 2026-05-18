---
layout: post
title: "Mesh 보충 3: Ambient Mode 깊게 — ztunnel + Waypoint"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio ambient platform senior-series supplement
---

8주차 Ambient를 내부 구조 + 운영 수준으로.

## 1. 두 layer 분리

기존 sidecar: L4 + L7 한 Envoy에서.
Ambient: L4 (ztunnel) + L7 (waypoint) 분리. 점진 가능.

## 2. ztunnel

- per node DaemonSet.
- Rust + Envoy 일부 코드.
- 책임: mTLS (HBONE 프로토콜) + L4 정책.

```
Pod A (no sidecar)
  ↓ TCP
[node A ztunnel] ─[HBONE mTLS over CONNECT]→ [node B ztunnel]
                                                    ↓
                                                  Pod B
```

HBONE = HTTP-Based Overlay Network Environment. CONNECT method로 L4 tunnel.

## 3. Waypoint

- per namespace/service Deployment.
- Envoy 기반.
- 책임: L7 정책 (HTTPRoute, AuthorizationPolicy L7).

```
Pod A → ztunnel (A) → ztunnel (B) → waypoint (B's namespace) → Pod B
```

L7 정책 없는 namespace는 waypoint 없음 → ztunnel 만으로.

## 4. 자원

- ztunnel: per node ~ 100MB.
- waypoint: per namespace ~ 50~100MB.
- sidecar (기존): per pod ~ 100MB.

50 Pod namespace 비교:
- sidecar: 50 × 100MB = 5GB.
- ambient: ztunnel 100MB + waypoint 100MB = 200MB.

큰 cluster에서 결정적 차이.

## 5. 활성

```bash
kubectl label namespace payments istio.io/dataplane-mode=ambient
```

기존 sidecar Pod은 영향 없음. 신규 또는 restart 시 ambient mode.

L7 추가:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: payments-waypoint, namespace: payments }
spec:
  gatewayClassName: istio-waypoint
```

## 6. 정책 차이

- PeerAuthentication: ztunnel이 처리.
- AuthorizationPolicy (L4): ztunnel.
- AuthorizationPolicy (L7): waypoint 필요.
- VirtualService / HTTPRoute: waypoint.

L4만 필요 한 namespace는 waypoint 없이.

## 7. Sidecar ↔ Ambient 마이그레이션

같은 cluster 혼합 가능:
```bash
# namespace A: sidecar (그대로)
# namespace B: ambient
```

점진 이주.

## 8. 한계

1. 일부 기능 미지원 (현재).
2. waypoint 추가 시 L7 hop +1.
3. ztunnel single point per node.
4. Operator support 추적 중.

## 9. 도입 평가

권장:
- Istio 1.22+ 이후 GA.
- 큰 cluster (sidecar 메모리 부담 큼).
- L7 정책 적은 namespace 우선.

보류:
- 작은 cluster (절감 미미).
- L7 정책 풍부 (waypoint 부담).

## 10. 결론

Ambient는 mesh 미래. Sidecar 자원 부담의 해법. 다만 신규 → 검증 후 도입. 빅테크 점진 채택 중.
