---
layout: post
title: "Mesh Week 10: 운영 — Upgrade, Performance, Troubleshooting"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series
---

## 학습 목표

- Canary upgrade로 무중단 istiod 업그레이드.
- Envoy 성능 튜닝.
- 흔한 troubleshooting 사례.

## 1. Canary Upgrade

Istio 표준:
1. 새 버전 istiod 별도 deploy (canary tag).
2. namespace에 새 tag 라벨.
3. Pod rollout으로 새 istiod 사용 sidecar 주입.
4. 모든 namespace 이주 후 옛 istiod 제거.

```bash
istioctl install --set revision=1-22-0
kubectl label namespace payments istio.io/rev=1-22-0 --overwrite
kubectl rollout restart deploy -n payments
```

revision 라벨로 점진 이주.

## 2. Performance 튜닝

### Sidecar 메모리
- Sidecar resource로 service catalog 제한.
- `maxConcurrentStreams` 등 Envoy 설정.

### CPU
- Envoy worker thread = CPU count.
- `concurrency` 설정.

### Telemetry 줄임
- trace sampling 1%.
- metric label cardinality 관리.

자세한 tuning은 [보충 4: Performance].

## 3. Troubleshooting 사례

### 사례 1: Pod이 mesh에 없음
- namespace `istio-injection=enabled` 확인.
- Pod restart.

### 사례 2: 503 UC (Upstream Connection error)
- target Pod 실제 ready인지.
- DestinationRule connectionPool 한계.

### 사례 3: 503 NR (No Route)
- VirtualService gateway 매칭.
- host 명 정확성.

### 사례 4: mTLS 실패
- PeerAuthentication mode.
- 양쪽 sidecar 모두 inject?

### 사례 5: 모든 트래픽 500
- istiod 자체 down?
- DestinationRule outlier 너무 강해 모두 eject?

## 4. 디버깅 도구

```bash
istioctl analyze              # 설정 분석
istioctl proxy-config <pod>   # 해당 Pod의 Envoy 설정
istioctl proxy-status         # 모든 Pod의 sync 상태
istioctl ps                   # 짧은 형식
kubectl logs -n istio-system <istiod-pod>
```

## 5. Cluster 부하

수천 Pod의 mesh:
- istiod CPU/메모리 폭증.
- xDS push 빈도.

대책:
- istiod scale (replica + HPA).
- Sidecar resource로 catalog 제한.
- ConfigMap 변경 빈도 줄임.

## 6. 운영 표준

- Istio 분기 1회 minor upgrade.
- canary upgrade로 무중단.
- 분기 1회 game day (mesh 장애 시뮬레이션).

## 7. 빅테크 사례

### Salesforce
자체 Istio 운영팀. version-pinning. PR 직접 contribute.

### Adobe
HelmRelease로 GitOps + canary revision 자동.

## 8. 자가평가 퀴즈

### Q1. Canary upgrade 핵심?
1. **revision 라벨로 점진 이주 — 무중단**
2. 한 번에 전체
3. UI
4. 무관

**정답: 1.**

### Q2. Sidecar resource 가치?
1. **service catalog 제한 → Envoy 메모리**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. istioctl analyze 가치?
1. **설정 오류 사전 검증**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q4. 503 NR 흔한 원인?
1. **VirtualService host/gateway 매칭 오류**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q5. mTLS 실패 디버깅?
1. **양쪽 sidecar inject + PeerAuth mode 확인**
2. UI
3. 빠름
4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 11: Linkerd 비교 + 선택 기준].
