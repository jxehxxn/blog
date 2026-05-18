---
layout: post
title: "SC Week 6: in-toto — Supply Chain의 표준 attestation"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain in-toto platform senior-series
---

## 학습 목표

- in-toto framework의 의미.
- Layout / Step / Link / Attestation.
- SLSA와 in-toto의 관계.

## 1. in-toto란

NYU 시작. 공급망 단계별 무결성 검증 framework. SLSA가 채택.

핵심: **statement (statement + predicate)** 의 표준 형식.

## 2. Statement 구조

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{"name": "...", "digest": {"sha256": "..."}}],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": { ... }
}
```

- subject: 대상 artifact.
- predicateType: 어떤 정보 type인지 URI.
- predicate: 실제 메타데이터.

## 3. Predicate Type 종류

- `slsaprovenance`: 빌드 출처.
- `spdxjson`: SBOM.
- `cyclonedx`: SBOM (대체).
- `vex`: VEX (vuln exploitability).
- `link`: in-toto의 step 결과.
- `custom`: 사내 정의.

## 4. Layout (선언) + Link (실행 증명)

- **Layout**: 공급망의 step 정의 (DevOps, Test, Build, Sign).
- **Link**: 각 step 실행 결과.
- 검증: layout 따라 모든 link 존재 + 무결성 검증.

## 5. Cosign에서 사용

```bash
cosign attest --predicate sbom.json --type spdxjson <image>
```

cosign이 내부에서 in-toto statement 형식으로 wrap → 서명 → OCI에 push.

## 6. Verification

```bash
cosign verify-attestation --type spdxjson --key cosign.pub <image>
# JSON statement 출력
```

policy 적용:
```bash
cosign verify-attestation --policy policy.cue ...
```

## 7. CUE / Rego policy

attestation의 predicate 안에 특정 조건 검사.

```cue
predicateType: "https://slsa.dev/provenance/v1"
predicate: {
  buildDefinition: {
    buildType: "https://slsa-framework.github.io/github-actions/v1"
  }
  runDetails: {
    builder: { id: =~"^https://github.com/actions/runner.*" }
  }
}
```

CUE는 type-safe, validation 강력.

## 8. VEX (Vulnerability Exploitability eXchange)

"이 CVE는 우리 코드에 영향 없음" 같은 evidence. attestation으로 첨부.

scan 시 VEX 참조해 false alarm 줄임.

## 9. 운영 함정

1. predicate type 표준 안 따름 → tooling 미지원.
2. cosign 검증 시 policy 없음 → 단순 존재만 확인.
3. layout vs attestation 혼동.

## 10. 자가평가

### Q1. in-toto statement?
1. **subject + predicateType + predicate**
2. UI 3. 무관 4. 다른 형식

**정답: 1.**

### Q2. Predicate type 표준?
1. **slsaprovenance/spdx/cyclonedx/vex** 2. 임의 3. UI 4. 무관

**정답: 1.**

### Q3. cosign attest 역할?
1. **in-toto wrap + 서명 + OCI push**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. CUE/Rego policy 가치?
1. **predicate 조건 검증**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. VEX의 가치?
1. **vuln false alarm 줄임 — "영향 없음" evidence**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음

[Week 7: Tekton Chains 운영].
