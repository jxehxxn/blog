---
layout: post
title: "SC Week 12: 캡스톤 — Source-to-Prod Supply Chain 풀스택"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series capstone
---

## 시나리오

MegaCorp: 250 서비스, EO 14028 + SOC 2.

목표: SLSA L3 + admission verify + incident drill.

## 산출물

1. 아키텍처: signed commit → Tekton + Chains → cosign → admission verify.
2. SBOM 생성 + Dependency Track.
3. Vuln scan + SLA.
4. Tabletop drill 결과.
5. SOC 2 + EO 14028 매핑.

## 루브릭 (100)

40 아키텍처 + 30 운영 + 30 incident.

## 모범답안

3종 (스타트업/스케일업/엔터프라이즈) — 별도.

## 결론

senior supply chain 운영 즉시.
