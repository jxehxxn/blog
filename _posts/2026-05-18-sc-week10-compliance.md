---
layout: post
title: "SC Week 10: Compliance — EO 14028, NIST SSDF, SOC 2"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain compliance platform senior-series
---

## 학습 목표

- US Executive Order 14028의 요구.
- NIST SSDF (SP 800-218).
- SOC 2 매핑.
- 빅테크 audit prep.

## 1. EO 14028 (2021)

미국 연방 SW 공급 의무:
- SBOM 제출.
- 보안 설계 표준 (NIST SSDF).
- vulnerability disclosure 절차.

영향: 빅테크 + 정부 거래 회사.

## 2. NIST SSDF (Secure Software Development Framework)

4 그룹:
- PO (Prepare Organization)
- PS (Protect Software)
- PW (Produce Well-secured Software)
- RV (Respond to Vulnerabilities)

각 그룹 아래 specific practice.

## 3. SLSA 매핑

- SSDF PW.4 → SLSA L1+.
- SSDF PS.3 → cosign 서명.
- SSDF RV.1 → vulnerability disclosure.

SLSA Level 3이면 SSDF 대부분 충족.

## 4. SOC 2 매핑

| SOC 2 | 우리 통제 |
|-------|----------|
| CC6.6 | Image admission (Kyverno) |
| CC7.1 | Vulnerability scan + alert |
| CC7.5 | Incident response + recovery |
| CC8.1 | Change management (PR + signed commit) |

## 5. 자동 evidence

- Tekton Chains attestation → 7년 보존.
- Vulnerability scan SARIF → SIEM.
- Admission deny event → audit log.

매월 자동 compliance report 생성.

## 6. 감사인 질문

1. "image 서명 검증을 어떻게 강제?"
2. "SBOM 어디 있나?"
3. "CRITICAL CVE 대응 시간은?"
4. "build 시스템 격리는?"
5. "compromised key 대응 절차?"

각 답변 + evidence 위치 사전 준비.

## 7. 빅테크 사례

- Adobe: SOC 2 + ISO 27001 + FedRAMP.
- Google: SSDF self-attestation 공개.
- Microsoft: SBOM 정부 제출.

## 8. 자가평가

### Q1. EO 14028 핵심?
1. **연방 SW SBOM 의무** 2. UI 3. 무관 4. 가격

**정답: 1.**

### Q2. NIST SSDF 4 그룹?
1. **PO/PS/PW/RV** 2. UI 3. 무관 4. 1개

**정답: 1.**

### Q3. SLSA L3과 SSDF?
1. **대부분 충족** 2. 무관 3. UI 4. 빠름

**정답: 1.**

### Q4. SOC 2 CC8.1 매핑?
1. **PR + signed commit + audit**
2. UI 3. 비용 4. 무관

**정답: 1.**

### Q5. 자동 evidence?
1. **Chains + scan + admission 적재**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음

[Week 11: Incident response].
