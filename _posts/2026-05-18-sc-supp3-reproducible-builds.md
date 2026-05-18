---
layout: post
title: "SC 보충 3: Reproducible Builds"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series supplement
---

## Reproducible 의미

같은 source + 같은 환경 → bit identical output.

장점: 누구나 재빌드 + 결과 비교 → 비인가 변경 탐지.

## 어려운 이유

1. Timestamp (build time).
2. Random (UUID, ID).
3. 환경 (locale, timezone).
4. 컴파일러 비결정성.
5. Network access (외부 fetch).

## 대응

1. SOURCE_DATE_EPOCH 환경변수.
2. Random seed 고정 또는 제거.
3. Sorted output.
4. Hermetic build (network off).
5. Locked deps (lock file).

## 도구

- Bazel: hermetic 강제.
- Nix: 표준 reproducible.
- diffoscope: 차이 분석.

## 운영

- CI에서 2번 빌드 후 hash 비교.
- 차이 있으면 fail.

## 결론

SLSA L4의 핵심 요구. 도달은 어려우나 핵심 component(예: kernel, OpenSSL) 부터 시작.
