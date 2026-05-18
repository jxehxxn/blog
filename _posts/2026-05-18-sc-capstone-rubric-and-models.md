---
layout: post
title: "SC Capstone: Rubric + 3종 모범답안"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series capstone
---

## 루브릭

40 아키텍처 + 30 운영 + 30 incident.

## 모범답안 A — 스타트업

- Trivy CI + cosign static key.
- SBOM 생성만, 검색 없음.
- 정책 admission 없음.

점수: ~50.

## 모범답안 B — 스케일업

- Tekton + Chains (SLSA L3).
- Syft SBOM + Dependency Track.
- Kyverno admission verify.
- Tabletop drill 분기.

점수: ~85.

## 모범답안 C — 엔터프라이즈

- Private Sigstore + self-hosted Rekor.
- Multi-attestation (SBOM+SLSA+VEX).
- SLSA L3 + reproducible 시도.
- 정부 보고 절차.
- SOC 2 + EO 14028 매핑.

점수: ~95.

## 결론

빅테크 senior supply chain 표준.
