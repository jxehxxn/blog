---
layout: post
title: "Mesh Week 7: Gateway + Ingress + Egress"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series
---

## 학습 목표

- Istio Gateway vs Kubernetes Ingress.
- Gateway API와의 관계.
- Egress Gateway로 외부 트래픽 통제.
- 다중 hostname + TLS.

## 1. 비유 — "공장 정문"

Service mesh는 공장 내 모든 부서 간 통신을 다룹니다. 공장 정문(Gateway) = 외부 ↔ 내부 경계. 들어오는 트래픽(Ingress) + 나가는 트래픽(Egress) 모두 통제.

## 2. Istio Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata: { name: prod-gateway, namespace: istio-system }
spec:
  selector:
    istio: ingressgateway
  servers:
    - port: { number: 443, name: https, protocol: HTTPS }
      tls:
        mode: SIMPLE
        credentialName: prod-tls
      hosts:
        - "app.mycorp.com"
        - "api.mycorp.com"
```

VirtualService와 결합:
```yaml
spec:
  hosts: [app.mycorp.com]
  gateways: [istio-system/prod-gateway]
  http:
    - match: [{ uri: { prefix: /api } }]
      route:
        - { destination: { host: api.payments.svc.cluster.local } }
    - route:
        - { destination: { host: frontend.web.svc.cluster.local } }
```

## 3. K8s Ingress vs Istio Gateway

| 항목 | K8s Ingress | Istio Gateway |
|------|-------------|----------------|
| 표현력 | 기본 (host/path) | 풍부 (header, weight, mirror) |
| TLS | annotation | 명시적 |
| Mesh 통합 | 별도 | 자연 |
| Gateway API | 차세대 | Istio도 Gateway API 지원 (v1.15+) |

Istio 환경에서는 Gateway가 자연. 단 Gateway API 표준화로 점진 통합.

## 4. Egress

기본적으로 mesh 내 service만 통신. 외부 service는 별도 등록.

### Default deny
```yaml
# istio-system/meshConfig
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

이러면 ServiceEntry 등록된 외부만 호출 가능. 사일런트 외부 호출 차단.

### ServiceEntry
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata: { name: github-api }
spec:
  hosts: [api.github.com]
  ports: [{ number: 443, name: https, protocol: HTTPS }]
  resolution: DNS
  location: MESH_EXTERNAL
```

## 5. Egress Gateway

외부로 나가는 트래픽도 별도 gateway 통과:

```
[App Pod] → [App sidecar] → [Egress Gateway] → External
```

장점:
- 외부 IP를 Egress Gateway IP로 통일 (allowlist 가능).
- 외부 트래픽 audit 집중.
- 외부 TLS 종결.

```yaml
kind: Gateway
metadata: { name: egress-github }
spec:
  selector: { istio: egressgateway }
  servers:
    - port: { number: 443, name: tls, protocol: TLS }
      hosts: [api.github.com]
      tls: { mode: PASSTHROUGH }
```

```yaml
kind: VirtualService
spec:
  hosts: [api.github.com]
  gateways: [egress-github, mesh]
  tls:
    - match: [{ port: 443, sniHosts: [api.github.com] }]
      route: [{ destination: { host: istio-egressgateway.istio-system.svc.cluster.local, port: { number: 443 } } }]
```

## 6. Multi-tenancy

각 팀이 자기 Gateway 별도 운영:
- payments-gateway: payments-api 만.
- web-gateway: frontend 만.

권한 분리, 변경 영향 격리.

## 7. cert-manager 통합

자동 TLS:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [app.mycorp.com]
      secretName: prod-tls   # cert-manager가 자동 생성
```

## 8. 운영 함정 5선

1. outboundTrafficPolicy ALLOW_ANY → 사일런트 외부 호출.
2. Gateway TLS credentialName 누락.
3. host 매칭 충돌 (여러 Gateway/VS).
4. Egress Gateway 미사용 → 외부 트래픽 추적 어려움.
5. ingress IP 노출 (cert-manager에 안 알려져) → TLS 발급 실패.

## 9. 실습

```bash
# 1. Gateway + VirtualService로 app.mycorp.com 외부 노출
# 2. cert-manager 자동 TLS
# 3. ServiceEntry로 외부 api.github.com 등록
# 4. Egress Gateway로 외부 호출 격리
# 5. outboundTrafficPolicy REGISTRY_ONLY → 비등록 외부 호출 차단 확인
```

## 10. 자가평가 퀴즈

### Q1. Istio Gateway vs K8s Ingress?
1. **Gateway가 mesh와 자연 통합 + 더 풍부**
2. 같음
3. 반대
4. 무관

**정답: 1.**

### Q2. outboundTrafficPolicy REGISTRY_ONLY 효과?
1. **ServiceEntry 등록된 외부만 호출 가능 — 사일런트 차단**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. Egress Gateway 가치?
1. **외부 IP 통일 + audit 집중**
2. UI
3. 무관
4. 빠름

**정답: 1.**

### Q4. cert-manager 통합?
1. **TLS Secret 자동 발급/갱신**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Multi-tenancy gateway?
1. **팀별 별도 gateway — 권한/영향 격리**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 8: Ambient Mode]에서 sidecar 없는 mesh.
