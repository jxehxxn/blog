---
layout: post
title: "SC Week 9: Source-to-Prod 풀스택 흐름"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series
---

## 학습 목표

- 코드 commit ~ production 까지 전체 supply chain.
- 각 단계별 보안 통제.
- E2E evidence chain.

## 1. Full Flow

```
[Developer commit (signed)]
   ↓
[GitHub branch protection + CODEOWNERS]
   ↓
[CI builds (Tekton in hosted runner)]
   ↓
[Trivy scan + SBOM (Syft) + SLSA provenance]
   ↓
[Cosign sign + Rekor 적재]
   ↓
[OCI registry push]
   ↓
[Image Updater PR to manifests repo]
   ↓
[Manifest PR review + merge]
   ↓
[ArgoCD sync]
   ↓
[K8s admission (Kyverno cosign verify)]
   ↓
[Pod running]
```

각 단계가 무결성 통제.

## 2. 단계별 통제

| 단계 | 통제 |
|------|------|
| Commit | GPG signed, 2FA |
| PR | required reviewer, CI green |
| Build | hosted, isolated |
| Scan | CRITICAL/HIGH 차단 |
| Sign | cosign keyless |
| Push | OCI immutable |
| Deploy PR | review |
| Admission | verify all attestations |

## 3. Evidence Chain

각 단계의 evidence가 다음 단계에 연결:
- Git commit signature → SLSA provenance subject.
- Build provenance → cosign attestation.
- Attestation → admission verify input.

## 4. 한 사고의 추적

가설: production image가 의심됨.
1. image digest로 cosign verify → signer 식별.
2. provenance에서 git commit + workflow run.
3. workflow run logs → 사용된 dep.
4. SBOM → 정확한 lib 버전.
5. Rekor에서 서명 시각 확인.

10분 안에 누가 무엇을 어떻게 만들었는지 추적.

## 5. 외부 의존성 처리

OSS lib: 자체 mirror + 자체 서명.
- Renovate로 자동 업데이트.
- 사내 OSS approval list.
- license audit.

## 6. Compromised Key 대응

cosign key 유출 → 새 key 발급 + Rekor 검색으로 영향 범위.

Keyless 사용하면 key 자체 없음 → 위험 감소.

## 7. 운영 함정

1. 단계 중 evidence 누락 → chain 깨짐.
2. 외부 dep 처리 없음.
3. emergency 우회 절차 부재.
4. evidence 보존 기간 짧음.

## 8. 자가평가

### Q1. E2E evidence chain 가치?
1. **사고 시 10분 추적**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. 외부 OSS 처리?
1. **사내 mirror + 자체 서명 + license audit**
2. 무시 3. UI 4. 무관

**정답: 1.**

### Q3. Compromised key 대응?
1. **새 key + Rekor로 영향 범위**
2. 무관 3. UI 4. 빠름

**정답: 1.**

### Q4. Emergency 우회?
1. **사전 정의 절차 + audit**
2. 직접 변경 3. UI 4. 무관

**정답: 1.**

### Q5. Keyless 장점 (key 유출)?
1. **key 자체 없음 → 위험 감소**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음

[Week 10: Compliance].
