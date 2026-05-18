---
layout: post
title: "Supply Chain Security 깊이 알기 (Senior Series 7/19)"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain sbom slsa cosign platform senior-series
---

## 강의 개요

빅테크 Senior 수준의 공급망 보안 12주. SBOM/SLSA/Cosign/Sigstore 모두.

전제 비유: **공급망 = 식품 유통**. 농장→가공→유통→매장→소비자. 어디서든 오염되면 끝. 식약처(SLSA)가 각 단계 표준 정함.

## 사전 지식

- Linux 셸, Docker, K8s 기초
- Git, CI/CD 기본 (Course 3)
- 보안 기초

## 학습 결과

12주 후: SBOM 생성·관리, 취약점 scan 자동화, image 서명 + verify, SLSA Level 3+ 구축, US EO 14028 / SOC 2 매핑, 인시던트 대응.

## 주차

| 주 | 주제 |
|----|------|
| 1 | 오리엔테이션 - 공급망 공격 사례 |
| 2 | SBOM (Syft, SPDX, CycloneDX) |
| 3 | Vulnerability scan (Trivy, Grype, Snyk) |
| 4 | Cosign + Sigstore + Rekor |
| 5 | SLSA framework 깊게 |
| 6 | in-toto + provenance |
| 7 | Tekton Chains 운영 |
| 8 | Admission verify |
| 9 | Source-to-prod 풀스택 |
| 10 | Compliance (EO 14028, NIST) |
| 11 | Incident response |
| 12 | Capstone |

## 보충
1. Sigstore 내부 (Fulcio, Rekor)
2. Dep confusion / typosquatting
3. Reproducible builds
4. Bill of attestations

## 다음

[Week 1]에서 SolarWinds, Log4Shell 같은 사례.
