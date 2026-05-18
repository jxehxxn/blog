---
layout: post
title: "SC 보충 2: Dependency Attacks — Confusion, Typosquatting, Takeover"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series supplement
---

## 1. Dependency Confusion

private package 이름이 public registry에 없으면, 같은 이름 public package를 자동 fetch 가능. 공격자가 public에 같은 이름 + 높은 버전 publish.

대응:
- 사내 registry priority.
- scoped package name (npm @mycorp/).
- explicit version.

## 2. Typosquatting

`requests` → `reqeusts` 같은 오타 패키지. CI/IDE auto-install로.

대응:
- approve list.
- pre-commit으로 typo 검사.

## 3. Maintainer Takeover

OSS maintainer 계정 인수 (xz utils case).

대응:
- 의존성 다양화 (1인 maintainer 의존 줄임).
- 정기 review.

## 4. Malicious Update

선의 lib가 갑자기 악성 업데이트.

대응:
- 자동 update 금지 (Renovate but with review).
- SBOM 변경 모니터링.

## 5. 결론

Dep attack은 가장 흔한 공급망 사고. SBOM + scan + review가 기본.
