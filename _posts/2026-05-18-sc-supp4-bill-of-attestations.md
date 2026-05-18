---
layout: post
title: "SC 보충 4: Bill of Attestations — Beyond SBOM"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series supplement
---

## 1. SBOM의 한계

SBOM은 "무엇이 들어 있나"만. 추가로 필요:
- 어떻게 빌드됐나 (SLSA provenance).
- 보안 테스트 결과 (VEX).
- 라이선스 (SPDX).
- 코드 리뷰 (signed commit).

이들을 묶은 "bill of attestations" 개념.

## 2. 예시

한 image에 첨부:
- SBOM (CycloneDX).
- SLSA provenance.
- VEX (vuln 영향).
- Trivy scan report.
- License audit.

모두 cosign attestation으로.

## 3. 검증

Kyverno 또는 cosign verify-attestation:

```yaml
verifyImages:
  - imageReferences: ["mycorp/*"]
    attestations:
      - predicateType: https://slsa.dev/provenance/v1
        attestors: [...]
      - predicateType: https://spdx.dev/Document
        attestors: [...]
      - predicateType: https://cyclonedx.org/schema
        attestors: [...]
```

모든 attestation 존재 + 검증 통과해야.

## 4. 표준화 노력

OpenSSF Scorecard, GUAC (Graph for Understanding Artifact Composition). 여러 도구가 attestation 통합.

## 5. 결론

미래는 SBOM 단독이 아닌 multi-attestation. 시니어는 흐름 추적.
