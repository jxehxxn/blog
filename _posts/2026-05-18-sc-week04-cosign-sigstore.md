---
layout: post
title: "SC Week 4: Cosign + Sigstore + Rekor 깊게"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain cosign sigstore platform senior-series
---

## 학습 목표

- Sigstore 3 컴포넌트.
- Cosign 서명 + 검증 흐름.
- Keyless (OIDC) 서명.
- Rekor transparency log.

## 1. Sigstore = "Let's Encrypt for code signing"

3 컴포넌트:
- **Cosign**: 서명/검증 CLI.
- **Fulcio**: 단기 인증서 발급 (OIDC 기반).
- **Rekor**: tamper-resistant transparency log.

## 2. Cosign 기본

### Static Key
```bash
cosign generate-key-pair
cosign sign --key cosign.key mycorp/myapp:v1.0.0
cosign verify --key cosign.pub mycorp/myapp:v1.0.0
```

서명은 image 옆 OCI artifact (`.sig` tag)로 push.

### Keyless (OIDC)
```bash
# CI에서 (OIDC token 자동 발급)
cosign sign --yes mycorp/myapp:v1.0.0
```

Fulcio가 OIDC 검증 후 단기(10분) cert 발급. 그 cert로 서명. cert + sig 모두 Rekor에 적재.

검증:
```bash
cosign verify \
  --certificate-identity ci-bot@mycorp.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com \
  mycorp/myapp:v1.0.0
```

## 3. Rekor

immutable append-only ledger. 모든 서명 이벤트 영구 기록.

```bash
rekor-cli search --artifact mycorp/myapp:v1.0.0
rekor-cli get --uuid <uuid>
```

검증 시 Rekor 내포함 증명 (Signed Entry Timestamp).

장점: 서명 키 분실/도용도 사후 audit 가능.

## 4. Fulcio 단기 인증서

10분 유효 → 키 누출 위험 거의 0.

cert 안에 OIDC subject가 박힘 (예: GitHub Actions run URL). cert가 "누가 서명했는지" 의 증거.

## 5. Attestation

서명만이 아니라 메타데이터 첨부:

```bash
cosign attest --predicate sbom.json --type spdxjson mycorp/myapp:v1.0.0
cosign attest --predicate slsa.json --type slsaprovenance mycorp/myapp:v1.0.0
```

검증:
```bash
cosign verify-attestation --type slsaprovenance --key cosign.pub mycorp/myapp:v1.0.0
```

## 6. K8s 통합 (Kyverno)

Course 6 Week 8 참조. cosign 서명 없으면 admission 거부.

## 7. 운영 함정

1. static key 분실 → 재서명 불가.
2. Rekor 없는 서명 → 사후 audit 불가.
3. OIDC subject pattern 너무 느슨 → 위변조 위험.
4. cert chain 검증 누락.
5. private Sigstore (self-hosted)와 public 혼용.

## 8. Self-hosted Sigstore

대규모/오프라인 환경. Fulcio + Rekor 자체 호스팅. 빅테크 일부 (Google, Adobe).

## 9. 자가평가

### Q1. Sigstore 3 컴포넌트?
1. **Cosign / Fulcio / Rekor** 2. UI 3. 무관 4. SBOM/SLSA/Slack

**정답: 1.**

### Q2. Keyless 장점?
1. **정적 키 부담 0, 10분 cert** 2. 빠름 3. UI 4. 무관

**정답: 1.**

### Q3. Rekor 가치?
1. **immutable audit ledger** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Attestation vs Signature?
1. **attestation은 메타데이터(SBOM, SLSA) 첨부**
2. 같음 3. UI 4. 무관

**정답: 1.**

### Q5. OIDC subject 검증?
1. **cert subject pattern 정확히 명시** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## 10. 다음

[Week 5: SLSA framework].
