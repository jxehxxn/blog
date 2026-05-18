---
layout: post
title: "Mesh Week 8: Ambient Mode — Sidecar 없는 Mesh"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio ambient platform senior-series
---

## 학습 목표

- Sidecar mode의 한계.
- Ambient 구조 — ztunnel + waypoint.
- L4/L7 분리.
- 도입 평가.

## 1. 비유 — "개인 비서 vs 공용 비서실"

Sidecar mode: 각 Pod 옆에 전용 비서(Envoy). 모든 Pod에 비서 1명 → 비용 큼.

Ambient mode: 노드 한 명 (ztunnel) + 부서별 비서실 (waypoint). 자원 효율.

## 2. Sidecar의 한계

- Pod마다 Envoy → CPU/메모리 증가 (보통 100m/100Mi per pod).
- Pod 재시작 = Envoy 재시작 → 잠시 트래픽 손실.
- 모든 Pod에 mesh inject 강제.

## 3. Ambient 구조

```
[ Pod ] → (iptables → uds → ztunnel) → 다른 노드 ztunnel → [ Pod ]
                                  ↓
                            (선택) waypoint proxy (L7)
```

- **ztunnel** (per node DaemonSet): L4 mTLS만. 가벼움 (Rust).
- **waypoint** (per namespace/service Deployment): L7 정책 (필요할 때만).

L4 (인증·암호화)는 무조건. L7 (route, retry, RBAC)는 필요한 namespace만 waypoint 추가.

## 4. 장점

1. Pod 자원 부담 0 (sidecar 제거).
2. Pod 재시작 영향 없음.
3. L7 필요한 곳만 waypoint → 점진 도입.
4. legacy app (sidecar injection 불가) 도 mesh 가능.

## 5. 단점

1. 신규 (v1.18+) → 안정성 추적 중.
2. waypoint 추가 시 L7 route 한 hop 늘어남.
3. 일부 기능 제한 (현재).
4. ztunnel 장애가 노드 영향.

## 6. 설치

```bash
istioctl install --set profile=ambient
```

namespace 활성:
```bash
kubectl label namespace payments istio.io/dataplane-mode=ambient
```

L7 필요:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: payments-waypoint, namespace: payments }
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - { name: mesh, port: 15008, protocol: HBONE }
```

waypoint Pod 자동 생성.

## 7. Sidecar ↔ Ambient 마이그레이션

namespace 단위로 점진:
1. 모든 namespace sidecar mode.
2. 한 namespace ambient label.
3. 검증.
4. 전환.

같은 cluster에 둘 다 가능.

## 8. 빅테크 평가

도입 시점:
- v1.20+ stable 후 본격 도입.
- 큰 cluster (sidecar 자원 부담 큼) 우선.
- L7 정책 적은 namespace에 먼저.

## 9. 운영 함정

1. ztunnel single point — node 자원 부족.
2. waypoint scale 안 함 → L7 병목.
3. policy 호환 (일부 PeerAuth 동작 다름).
4. legacy 일부 traffic 동작 변경.

## 10. 자가평가 퀴즈

### Q1. Sidecar mode 한계?
1. **Pod 자원 부담 + 재시작 영향**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q2. Ambient의 ztunnel 역할?
1. **per node L4 mTLS proxy**
2. L7
3. UI
4. 무관

**정답: 1.**

### Q3. Waypoint 사용?
1. **L7 정책 필요한 namespace만 추가**
2. 모든 곳
3. 무관
4. UI

**정답: 1.**

### Q4. Ambient 마이그레이션?
1. **namespace 단위 점진**
2. 한 번에 전체
3. 무관
4. UI

**정답: 1.**

### Q5. Ambient의 약점?
1. **신규 — 안정성 추적 중**
2. 안전
3. 빠름
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 9: Multi-cluster mesh].
