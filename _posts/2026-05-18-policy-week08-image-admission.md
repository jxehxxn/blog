---
layout: post
title: "Policy Week 8: Image Admission + 공급망 보안"
date: 2026-05-18 20:00:00 +0900
categories: policy kyverno cosign supply-chain platform senior-series
---

## 학습 목표

- VerifyImages로 cosign 서명 검증.
- Attestation (SLSA provenance) 검증.
- Image policy 패턴.
- 공급망 전체 흐름.

## 1. 비유 — "수입품 검사"

cluster에 들어오는 모든 image를 정문에서 검사. 서명·출처 증명 없으면 거부.

## 2. Kyverno verifyImages 기본

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-cosign }
spec:
  validationFailureAction: enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: verify-mycorp-images
      match:
        any: [{ resources: { kinds: [Pod] } }]
      verifyImages:
        - imageReferences: ["mycorp/*"]
          mutateDigest: true
          attestors:
            - entries:
                - keys:
                    publicKeys: |
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

서명 미검증 image는 admission 거부.

`mutateDigest: true`: image tag를 digest로 자동 변환 (immutable).

## 3. Keyless (Sigstore)

```yaml
attestors:
  - entries:
      - keyless:
          issuer: https://token.actions.githubusercontent.com
          subject: "https://github.com/mycorp/myapp/.github/workflows/release.yml@refs/heads/main"
```

GitHub Actions OIDC 기반 서명. 정적 key 관리 부담 0.

## 4. Attestation 검증 (SLSA)

서명만이 아니라 build provenance 검증.

```yaml
verifyImages:
  - imageReferences: ["mycorp/*"]
    attestations:
      - predicateType: https://slsa.dev/provenance/v1.0
        attestors:
          - entries:
              - keyless:
                  issuer: https://token.actions.githubusercontent.com
                  subject: "https://github.com/mycorp/build/..."
        conditions:
          - all:
              - key: "{{ predicate.buildType }}"
                operator: Equals
                value: "https://slsa-framework.github.io/github-actions/v1"
```

build 시스템 + branch + workflow 검증. 임의 push로 만든 image 차단.

## 5. SBOM 검증

```yaml
attestations:
  - predicateType: https://spdx.dev/Document
    conditions: ...
```

SBOM 첨부 의무화.

## 6. Image Registry 제한

```yaml
validate:
  message: "untrusted registry"
  pattern:
    spec:
      containers:
        - image: "mycorp.io/* | gcr.io/google-containers/*"
```

화이트리스트 registry만.

## 7. Image Tag 정책

```yaml
validate:
  message: "no :latest"
  pattern:
    spec:
      containers:
        - image: "!*:latest"
```

`:latest` 거부. immutable 강제.

## 8. 공급망 전체 흐름

```
[Source code] → [build (Tekton)] → [cosign sign + SLSA attest]
       ↓
[OCI registry]
       ↓
[K8s Pod create]
       ↓
[Kyverno verifyImages: signature + attestation 검증]
       ↓ pass
[Pod runs]
```

## 9. 빅테크 사례

### Shopify
모든 internal image cosign keyless 서명. Kyverno admission으로 unsigned 차단.

### Google
사내 모든 image SLSA Level 3. policy로 강제.

## 10. 운영 함정

1. 서명만 검증, attestation 미검증 → SLSA Level 2까지만.
2. mutateDigest 미설정 → tag-based image 변경 위험.
3. webhook timeout 짧음 → 검증 fail.
4. 외부 (3rd-party) image policy 부재 → 외부 image 미검증.
5. 서명 키 노출 → 의미 없음.

## 11. 자가평가 퀴즈

### Q1. VerifyImages 효과?
1. **cosign signature 검증 + unsigned 거부**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Keyless 가치?
1. **OIDC 기반 → 정적 key 관리 부담 0**
2. UI 3. 비용 4. 무관

**정답: 1.**

### Q3. mutateDigest?
1. **tag → digest 변환 (immutable)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. SLSA attestation 검증?
1. **build provenance까지 검증 → SLSA Level 3**
2. signature만 3. UI 4. 무관

**정답: 1.**

### Q5. `:latest` 거부?
1. **immutable tag 강제 — drift 방지**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 12. 다음

[Week 9: Audit + Violations].
