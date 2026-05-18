---
layout: post
title: "SC Week 8: Admission Verify — Cluster 진입 게이트"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain admission platform senior-series
---

## 학습 목표

- Kyverno/Connaisseur/Sigstore Policy Controller 비교.
- 서명 + SBOM + SLSA 모두 검증.
- 정책 점진 도입.

## 1. 비유 — "공항 게이트"

Cluster에 들어오는 image를 보안 검사:
- 여권 확인 (cosign signature)
- 짐 검사 (SBOM)
- 출국 기록 (SLSA provenance)

## 2. Kyverno (가장 광범위)

Course 6 Week 8 참조.

```yaml
spec:
  rules:
    - name: verify
      verifyImages:
        - imageReferences: ["mycorp/*"]
          attestors: [...]
          attestations:
            - predicateType: https://slsa.dev/provenance/v1
              attestors: [...]
              conditions:
                - all:
                    - { key: "{{ predicate.buildDefinition.buildType }}", operator: Equals, value: "..." }
```

## 3. Sigstore Policy Controller

K8s admission controller. Sigstore 전용.

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata: { name: mycorp-policy }
spec:
  images:
    - glob: "mycorp/*"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://accounts.google.com
            subject: ci-bot@mycorp.iam
```

Kyverno 보다 가벼움, Sigstore에 특화.

## 4. Connaisseur

다른 admission controller. notary/cosign 모두 지원.

## 5. 정책 점진 도입

```
[ Phase 1 ]: warn only (audit log)
[ Phase 2 ]: enforce 일부 namespace
[ Phase 3 ]: 전체 enforce
[ Phase 4 ]: SLSA + SBOM 검증 추가
```

## 6. 외부 image 처리

3rd-party image (nginx 등) 은 직접 서명 안 됨. 대안:
- 사내 mirror에 cosign 서명.
- 또는 trusted publisher list (Docker Official, Quay verified 등).
- 또는 specific image hash allowlist.

## 7. Performance

verifyImages는 외부 (Rekor) 호출 → latency. cache + parallel.

## 8. 운영 함정

1. policy 너무 strict → 정상 image도 차단 (운영 마비).
2. external image 처리 정책 부재.
3. CDN/Rekor 장애 시 admission fail.
4. policy 변경 audit 부재.
5. exemption 영구.

## 9. 자가평가

### Q1. 가장 광범위 admission 도구?
1. **Kyverno** 2. Connaisseur 3. Sigstore policy 4. 무관

**정답: 1.**

### Q2. 정책 점진?
1. **warn → enforce 일부 → 전체**
2. 한 번에 3. UI 4. 무관

**정답: 1.**

### Q3. 외부 image 처리?
1. **사내 mirror + 서명 또는 allowlist**
2. 무시 3. UI 4. 무관

**정답: 1.**

### Q4. Rekor 장애 영향?
1. **admission fail 가능 → fallback 정책**
2. 무관 3. UI 4. 빠름

**정답: 1.**

### Q5. Phase 4의 검증?
1. **SLSA provenance + SBOM**
2. signature만 3. UI 4. 무관

**정답: 1.**

## 10. 다음

[Week 9: Source-to-prod 풀스택].
