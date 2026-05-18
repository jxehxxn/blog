---
layout: post
title: "Mesh 보충 1: Envoy 깊게 — xDS, Filter Chain, HTTP Connection Manager"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh envoy platform senior-series supplement
---

2주차 Envoy를 내부 구조 수준으로.

## 1. xDS 동작

```
istiod -[gRPC ADS stream]→ Envoy
        - LDS update
        - RDS update
        - CDS update
        - EDS update
        - SDS update
```

ADS (Aggregated Discovery Service) 단일 stream으로 모두. v3 API.

## 2. Filter Chain

Envoy의 본질: chain of filter.

```
TCP listener
  ↓
Network Filters (read/write byte stream)
  - TLS termination
  - HTTP Connection Manager (HTTP 파싱)
  ↓
HTTP Filters
  - Router (라우팅)
  - Fault injection
  - Rate limit
  - Lua/Wasm 커스텀
  ↓
Upstream cluster
```

각 filter가 단일 책임. 조합으로 임의 동작.

## 3. HTTP Connection Manager

가장 자주 쓰는 filter. HTTP 요청 종결 + 처리.

```yaml
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": ...
    codec_type: AUTO   # HTTP/1.1, HTTP/2 자동
    route_config: ...
    http_filters:
      - name: envoy.filters.http.router
```

## 4. Cluster

Upstream service 그룹. Load balancing 정책, health check.

```yaml
clusters:
  - name: productpage
    type: STRICT_DNS
    load_assignment:
      endpoints:
        - lb_endpoints:
            - endpoint: { address: { socket_address: { address: productpage, port_value: 9080 } } }
    health_checks:
      - http_health_check: { path: /healthz }
```

## 5. Endpoint Discovery (EDS)

Cluster의 endpoint는 동적. istiod가 K8s service의 endpoint를 EDS로 push.

```
istiod → Envoy: "productpage cluster의 endpoints: 10.0.0.5:9080, 10.0.0.6:9080"
Pod 죽으면 istiod가 즉시 새 endpoint 목록 push.
```

## 6. Wasm Filter

Envoy 확장. Rust/AssemblyScript로 짠 wasm 모듈을 동적 load.

```yaml
http_filters:
  - name: envoy.filters.http.wasm
    typed_config:
      config:
        vm_config:
          runtime: envoy.wasm.runtime.v8
          code: { local: { filename: "/etc/envoy/my-filter.wasm" } }
```

사용: 사내 custom logic (특수 헤더, custom auth).

## 7. Lua Filter

가벼운 case는 Lua 인라인:
```yaml
http_filters:
  - name: envoy.filters.http.lua
    typed_config:
      inline_code: |
        function envoy_on_request(handle)
          handle:headers():add("x-injected", "true")
        end
```

## 8. Listener의 listen address

iptables redirect로 모든 outgoing TCP가 `0.0.0.0:15001` Envoy listener에. 내부 routing으로 적절 cluster.

## 9. 운영 도구

```bash
istioctl proxy-config listeners <pod>
istioctl proxy-config routes <pod>
istioctl proxy-config clusters <pod>
istioctl proxy-config endpoints <pod>
istioctl proxy-config secret <pod>
```

각 Envoy의 LDS/RDS/CDS/EDS/SDS 상태 확인. 디버깅의 핵심.

## 10. 결론

Envoy는 단순한 proxy가 아니라 filter chain 기반 일반 통신 platform. 시니어는 xDS, filter, wasm 까지 이해해야 진짜 mesh 디버깅 가능.
