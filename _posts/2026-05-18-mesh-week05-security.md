---
layout: post
title: "Mesh Week 5: 보안 — mTLS, AuthorizationPolicy, JWT, Zero-Trust"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio security platform senior-series
---

## 학습 목표

- 자동 mTLS의 동작 원리.
- PeerAuthentication 모드 (STRICT/PERMISSIVE/DISABLE).
- AuthorizationPolicy로 L7 인가.
- RequestAuthentication으로 JWT 검증.

## 1. 비유 — "사내 신원증 + 출입 기록"

10명 부서원이 서로 신원증 확인 없이 출입 → 외부 침입자 구분 불가.
신원증 발급 (mTLS) + 출입 기록 + 부서별 접근 통제 (AuthorizationPolicy) = zero-trust.

## 2. 자동 mTLS

Istio가 모든 Pod 간 트래픽을 자동 mTLS:
- Citadel이 각 SA(ServiceAccount)에 SPIFFE 형식 인증서 발급 (24시간 자동 회전).
- Envoy가 TLS handshake.
- App 코드 변경 0.

## 3. PeerAuthentication

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: default, namespace: istio-system }
spec:
  mtls: { mode: STRICT }
```

namespace 단위로도:
```yaml
metadata: { namespace: payments }
spec:
  mtls: { mode: STRICT }
```

모드:
- **STRICT**: mTLS만.
- **PERMISSIVE**: plaintext도 허용 (마이그레이션 중).
- **DISABLE**: mTLS 끔.

빅테크 표준: cluster-wide PERMISSIVE → namespace별 STRICT 점진 전환.

## 4. AuthorizationPolicy

L7 인가. 누가 누구에게 무엇을 할 수 있나.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: payments-allow, namespace: payments }
spec:
  selector:
    matchLabels: { app: payments-api }
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/web/sa/web-frontend"]
      to:
        - operation:
            methods: [GET, POST]
            paths: ["/api/v1/charges/*"]
```

- web-frontend SA만 payments-api에 GET/POST 가능.
- 그 외는 deny.

deny by default 권장:
```yaml
spec:
  selector:
    matchLabels: { app: payments-api }
  action: DENY
  rules:
    - {}   # 모든 요청 deny
```

이후 위 ALLOW 규칙으로 화이트리스트.

## 5. RequestAuthentication (JWT)

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata: { name: payments-jwt }
spec:
  selector:
    matchLabels: { app: payments-api }
  jwtRules:
    - issuer: "https://accounts.mycorp.com"
      jwksUri: "https://accounts.mycorp.com/.well-known/jwks.json"
      audiences: ["payments-api"]
```

JWT 자동 검증. valid면 request에 attribute 추가. AuthorizationPolicy에서 사용:

```yaml
rules:
  - from:
      - source:
          requestPrincipals: ["https://accounts.mycorp.com/*"]
```

## 6. Zero-Trust 패턴

빅테크 표준:
1. Cluster mTLS STRICT.
2. namespace deny by default.
3. AllowList AuthorizationPolicy.
4. external request에 JWT 검증.
5. principal 기반 audit log.

## 7. CA 관리

istiod 내장 CA가 기본. 빅테크 표준:
- 사내 PKI (Vault PKI) 통합.
- Root CA 분리 → istiod는 intermediate.
- 인증서 회전 자동.

자세한 CA 운영은 [보충 2: mTLS deep].

## 8. 운영 함정 5선

1. STRICT 전환 시 외부 client (mTLS 없음) 실패.
2. AuthorizationPolicy ALLOW만 + DENY 없음 → 의미 약함.
3. JWT 검증 누락 → 위변조 토큰 통과.
4. CA root 노출.
5. principal 매핑 오해 (SA 이름).

## 9. 빅테크 사례

### Salesforce
모든 internal 통신 mTLS STRICT. compliance audit 통과.

### Adobe
Vault PKI + istiod 통합. 일관 PKI.

## 10. 실습

```bash
# 1. PeerAuthentication PERMISSIVE → STRICT
# 2. AuthorizationPolicy deny by default
# 3. ALLOW 규칙으로 web-frontend → payments-api
# 4. 다른 SA로 시도 → 거부 확인
# 5. RequestAuthentication으로 JWT 검증
```

## 11. 자가평가 퀴즈

### Q1. 자동 mTLS 인증서?
1. **SPIFFE 형식, 24시간 자동 회전**
2. 영구
3. UI
4. 무관

**정답: 1.**

### Q2. STRICT vs PERMISSIVE?
1. **STRICT mTLS만, PERMISSIVE plaintext도 허용**
2. 같음
3. 반대
4. 무관

**정답: 1.**

### Q3. AuthorizationPolicy의 zero-trust 패턴?
1. **deny by default + allow whitelist**
2. allow only
3. deny only
4. 무관

**정답: 1.**

### Q4. JWT 검증 위치?
1. **RequestAuthentication**
2. PeerAuthentication
3. DestinationRule
4. 무관

**정답: 1.**

### Q5. 빅테크 CA 패턴?
1. **사내 PKI (Vault) + istiod intermediate**
2. istiod 내장 root
3. 외부 CA만
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 6: 관측]에서 Kiali, Jaeger, Prometheus 통합.
