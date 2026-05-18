---
layout: post
title: "Mesh Week 9: Multi-cluster Mesh — Primary-Remote, Multi-Primary"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio multi-cluster platform senior-series
---

## 학습 목표

- Multi-cluster의 3가지 토폴로지.
- Network 모델 (single vs multi).
- East-west gateway.
- 멀티 클러스터 서비스 발견.

## 1. 비유 — "여러 캠퍼스 통합"

같은 대학의 5개 캠퍼스가 서로 학과 공유. 캠퍼스 내(같은 cluster)는 자유 통신. 캠퍼스 간(다른 cluster)은 정문(east-west gateway) 통과.

## 2. 토폴로지

### Primary-Remote
1 control plane (primary cluster) + N remote cluster (data plane만).

장점: 운영 단순.
단점: primary 장애 시 전체 영향.

### Multi-Primary
각 cluster에 control plane. 서로 trust.

장점: 격리.
단점: 운영 복잡.

### Multi-Network
다른 network (다른 VPC). east-west gateway로 연결.

## 3. 같은 Network vs 다른 Network

### 같은 network (flat)
Pod IP가 cluster 간 라우팅 가능. 가장 빠름. 같은 VPC.

### 다른 network
east-west gateway로 mTLS tunnel. cross-VPC, cross-cloud.

## 4. East-West Gateway

cluster 경계에서 trust 통신. SNI 기반 라우팅.

```yaml
kind: Gateway
metadata: { name: eastwest, namespace: istio-system }
spec:
  selector: { istio: eastwestgateway }
  servers:
    - port: { number: 15443, name: tls, protocol: TLS }
      tls: { mode: AUTO_PASSTHROUGH }
      hosts: ["*.local"]
```

## 5. Service Discovery

각 cluster의 service가 다른 cluster에서도 인지되어야:

```bash
istioctl create-remote-secret --name=cluster2 | kubectl apply -f -
```

cluster1의 istiod가 cluster2의 API server 접근 → cluster2의 service 인지.

## 6. Locality LB

Multi-cluster에서 같은 region 우선:
```yaml
spec:
  trafficPolicy:
    localityLbSetting:
      enabled: true
      distribute:
        - from: us-east-1/*
          to: { "us-east-1/*": 80, "us-west-1/*": 20 }
```

us-east-1 client는 us-east-1 endpoint 우선, us-west-1 fallback.

## 7. Multi-cluster DR

primary down 시 remote가 자체적으로 traffic 처리. 단 control plane 변경 불가.

multi-primary가 더 안전.

## 8. 빅테크 사례

### Salesforce
multi-region multi-primary. 각 region 독립.

### Adobe
primary-remote 시작 → multi-primary로 이주.

## 9. 운영 함정 5선

1. trust bundle 미공유 → mTLS 실패.
2. CIDR 충돌 → 같은 network에서 routing 실패.
3. east-west gateway HA 부재.
4. multi-primary CA 동기화 누락.
5. cross-cluster latency 무시 → 사용자 경험 저하.

## 10. 자가평가 퀴즈

### Q1. Primary-Remote vs Multi-Primary?
1. **Primary-Remote 단순, Multi-Primary 격리**
2. 같음
3. 무관
4. UI

**정답: 1.**

### Q2. East-west gateway의 가치?
1. **cluster 간 mTLS tunnel (다른 network)**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. trust bundle 공유?
1. **각 cluster의 CA를 서로 신뢰 — 없으면 mTLS 실패**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. Locality LB의 가치?
1. **같은 region 우선 → latency/cost**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q5. multi-primary 도입 이유?
1. **격리 + DR + control plane 독립**
2. 단순
3. UI
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 10: 운영]에서 upgrade, performance, troubleshooting.
