---
layout: post
title: "K8s Deep 보충 2: CNI 비교 — Calico vs Cilium vs Flannel"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series supplement networking
---

4주차에서 짚고 넘어간 CNI 3종을 정량 비교합니다.

## 1. 한 줄 요약

- **Flannel**: 가장 단순, NetworkPolicy 없음. 학습/소규모.
- **Calico**: BGP 라우팅 + 강력한 NetworkPolicy. 운영 표준.
- **Cilium**: eBPF 기반, L7 정책 + 관측성 + kube-proxy 대체. 차세대 표준.

## 2. 비교 표

| 항목 | Flannel | Calico | Cilium |
|------|---------|--------|--------|
| Data plane | VXLAN overlay | BGP routing (또는 IPIP/VXLAN) | eBPF |
| NetworkPolicy | 미지원 | 표준 + extended | 표준 + L7 |
| L7 정책 | X | X | O (HTTP/gRPC/Kafka) |
| 관측성 | 기본 | 일반 | Hubble (강력) |
| 성능 | 보통 | 높음 (native BGP 시) | 매우 높음 |
| kube-proxy 대체 | X | X | O |
| 운영 난이도 | 매우 쉬움 | 중간 | 중상 |
| 빅테크 사용 | 적음 | 많음 | 증가 추세 |

## 3. 비유 — "캠퍼스 우편"

- Flannel: 모든 우편물을 도시 우편국(VXLAN encap)으로 보냄. 단순하지만 우편국이 병목.
- Calico: 캠퍼스 우편 트럭이 직접 다른 캠퍼스로 (BGP 라우팅). 빠르고 직접적.
- Cilium: 첨단 자율 우편 시스템(eBPF). 우편물 안 내용까지 검사 가능(L7), 통계 자동 수집.

## 4. Calico 깊게

### BGP 모드
각 노드가 자신의 podCIDR을 BGP로 광고. 클러스터 외부 라우터가 podCIDR 트래픽을 알아서 라우팅. 가장 빠름.

요구: 물리망이 BGP 가능.

### IPIP / VXLAN 모드
물리망이 BGP 안 되면 encap 사용. overhead 발생.

### NetworkPolicy + GlobalNetworkPolicy
표준 NetworkPolicy 외에 Calico 전용 `GlobalNetworkPolicy` 로 cluster-wide 정책.

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata: { name: deny-egress-internet }
spec:
  selector: all()
  egress:
    - action: Deny
      destination:
        notNets: [10.0.0.0/8, 192.168.0.0/16]
```

## 5. Cilium 깊게

### eBPF
Linux 커널에 안전한 sandboxed program을 주입. iptables보다 빠르고 더 깊은 hook 가능.

### kube-proxy 대체
```bash
helm install cilium cilium/cilium --set kubeProxyReplacement=true
# kube-proxy 제거 가능
```

iptables 규칙 폭증 없이 Service 라우팅. 5000+ Service 환경에서 결정적 성능 차이.

### L7 정책

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: api-only-get }
spec:
  endpointSelector:
    matchLabels: { app: api }
  ingress:
    - fromEndpoints:
        - matchLabels: { app: web }
      toPorts:
        - ports: [{port: "80", protocol: TCP}]
          rules:
            http:
              - method: "GET"
                path: "/api/v1/.*"
```

L4뿐 아니라 HTTP method + path까지 검사.

### Hubble (관측)
모든 Pod-to-Pod 트래픽을 자동 기록. `hubble observe` 로 실시간 추적.

## 6. Flannel — 언제 쓸까

- 학습용 (kind/minikube 기본).
- 매우 작은 클러스터 + 보안 요구 약함.
- 운영에서는 NetworkPolicy 부재가 치명적 → 권장 안 함.

## 7. 빅테크 선택 패턴

- 전통: Calico (안정성·성숙도).
- 최신/대규모: Cilium (성능·관측성·L7).
- 일부 클라우드: AWS VPC CNI (EKS native), Azure CNI 등 클라우드 통합.

GKE는 Dataplane V2 = Cilium 기반.

## 8. 마이그레이션 위험

CNI 변경은 cluster를 거의 재구성 수준 → 신중. 처음 도입 시 잘 고르는 것이 best.

## 9. 결론

- 처음 학습 → Flannel OK.
- production → Calico 또는 Cilium.
- 미래 → Cilium 점점 표준.
