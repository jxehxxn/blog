---
layout: post
title: "SC 최종 자가평가 — 30문항 종합 (정답+해설)"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series assessment
---

12주 학습자 종합.

## Q1. 공급망 4 유형?
1. **Source/Build/Dep/Distribution** 2. UI 3. 무관 4. CPU

**정답: 1.**

## Q2. SLSA 4 레벨 핵심 차이?
1. **L1 provenance / L2 hosted+signed / L3 non-falsifiable / L4 hermetic+repro**
2. UI 3. 같음 4. 무관

**정답: 1.**

## Q3. SBOM 표준 2종?
1. **SPDX + CycloneDX** 2. UI 3. 무관 4. JSON+YAML

**정답: 1.**

## Q4. Syft 입력?
1. **container image/디렉토리** 2. URL 3. UI 4. 무관

**정답: 1.**

## Q5. Trivy 영역?
1. **container+source+IaC** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q6. CRITICAL SLA?
1. **24h patch** 2. 1년 3. UI 4. 무관

**정답: 1.**

## Q7. Sigstore 3 컴포넌트?
1. **Cosign/Fulcio/Rekor** 2. UI 3. 무관 4. SBOM/SLSA/Slack

**정답: 1.**

## Q8. Keyless 장점?
1. **10분 cert → 정적 key 부담 0** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q9. Rekor?
1. **immutable audit ledger** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q10. Attestation vs Signature?
1. **attestation은 메타데이터 첨부** 2. 같음 3. UI 4. 무관

**정답: 1.**

## Q11. SLSA L3 핵심?
1. **빌드 서비스가 provenance 서명** 2. 사용자 키 3. UI 4. 무관

**정답: 1.**

## Q12. Hermetic 의미?
1. **빌드 외부 네트워크 없음** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q13. in-toto statement?
1. **subject + predicateType + predicate** 2. UI 3. 무관 4. 다른

**정답: 1.**

## Q14. VEX 가치?
1. **vuln "영향 없음" evidence** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q15. Chains 자동?
1. **provenance + 서명 + Rekor 적재** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q16. 표준 결과 이름?
1. **IMAGE_URL + IMAGE_DIGEST** 2. UI 3. 무관 4. result

**정답: 1.**

## Q17. Kyverno verifyImages?
1. **cosign sig + attestation 검증** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q18. Sigstore Policy Controller?
1. **Sigstore 전용 admission** 2. Kyverno 3. UI 4. 무관

**정답: 1.**

## Q19. 외부 image 처리?
1. **사내 mirror + 서명 또는 allowlist** 2. 무시 3. UI 4. 무관

**정답: 1.**

## Q20. E2E evidence chain?
1. **각 단계 evidence 연결 → 10분 추적** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q21. EO 14028?
1. **연방 SW SBOM 의무** 2. UI 3. 무관 4. 가격

**정답: 1.**

## Q22. NIST SSDF 4 그룹?
1. **PO/PS/PW/RV** 2. UI 3. 무관 4. 1개

**정답: 1.**

## Q23. SOC 2 CC8.1 매핑?
1. **PR + signed commit + audit** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q24. CVE in dep 1차?
1. **SBOM 검색으로 영향 image** 2. 무시 3. UI 4. 무관

**정답: 1.**

## Q25. Compromised key + Rekor?
1. **시각으로 의심 서명 검색** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q26. Tabletop 가치?
1. **훈련 → 실전 빠름** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q27. Dep Confusion 대응?
1. **사내 registry priority + scoped name** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q28. Reproducible 어려운 이유?
1. **timestamp, random, network 등 비결정성** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q29. Bill of attestations?
1. **SBOM + SLSA + VEX + scan 통합** 2. SBOM 만 3. UI 4. 무관

**정답: 1.**

## Q30. 미래 supply chain?
1. **multi-attestation + 자동 검증** 2. SBOM 만 3. UI 4. 무관

**정답: 1.**

다음: ArgoCD 심화.
