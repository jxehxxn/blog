---
layout: post
title: "Mesh 보충 2: mTLS + CA 관리 깊게"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio security platform senior-series supplement
---

5주차 mTLS를 PKI 수준으로.

## 1. SPIFFE / SPIRE

Istio가 사용하는 ID 표준.

```
spiffe://cluster.local/ns/payments/sa/payments-api
```

= cluster + namespace + ServiceAccount → cluster 전역 유일.

이걸 X.509 인증서의 SAN(Subject Alternative Name) URI로 인코딩.

## 2. Citadel CA

istiod 내장 CA. self-signed root + intermediate.

문제: root key가 istiod Pod 안에 있음. 노출 위험.

## 3. 사내 PKI 통합

### 방법 1: Vault PKI
istiod가 Vault에서 intermediate 발급 → 자체 자식 cert 발급.

### 방법 2: 외부 root 주입
사내 PKI에서 root + intermediate 만들고 istiod에 secret으로 주입.

```bash
kubectl create secret generic cacerts -n istio-system \
  --from-file=ca-cert.pem \
  --from-file=ca-key.pem \
  --from-file=root-cert.pem \
  --from-file=cert-chain.pem
```

istiod 재시작 후 이 cert로 자식 cert 발급. 사내 root chain에 통합.

## 4. 인증서 라이프사이클

- 발급: SA 생성 시.
- 갱신: 24시간 (기본). cert 만료 12시간 전 자동.
- 회수: SA 삭제 또는 explicit revocation (드물).

## 5. mTLS Handshake 흐름

```
Client Envoy → Server Envoy
  1. TLS ClientHello (with SNI)
  2. Server ClientCertificateRequest
  3. Client → cert + signature
  4. Server verifies (root CA, SPIFFE ID)
  5. Server → cert + signature
  6. Client verifies
  7. Encrypted application data
```

Pod 입장: localhost로 plaintext, sidecar가 모든 처리.

## 6. STRICT vs PERMISSIVE 전환

production 점진:
1. cluster-wide PERMISSIVE.
2. namespace별 STRICT 전환.
3. 비 mesh 외부와 통신할 namespace는 PERMISSIVE 유지 또는 explicit allowance.

## 7. 외부 client (non-mesh)

브라우저, mobile app 등은 sidecar 없음. Ingress Gateway에서 표준 TLS 종결, 내부는 mTLS.

```
[Browser TLS] → [Ingress Gateway] → [mTLS] → [Pod sidecar] → [Pod app]
```

## 8. CA 비상

CA key가 노출되면? 모든 SA cert 회수. 비현실.

대책:
- root key를 HSM에.
- intermediate 자주 회전.
- root 사용 최소화 (intermediate가 일상).

## 9. 운영 함정

1. CA root cert 분실 → trust chain 깨짐.
2. intermediate 회전 시 trust bundle 동기화 누락 → 일시 통신 fail.
3. PERMISSIVE 영원히 → mTLS 의미 없음.
4. SPIFFE ID 오해 (SA 이름 기반).
5. external client cert 직접 검증 (잘못된 패턴).

## 10. 결론

mTLS는 mesh의 핵심 가치. CA 관리가 mesh 운영의 70%. 시니어는 사내 PKI 통합 + 회전 자동화 + 비상 절차까지.
