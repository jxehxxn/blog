---
layout: post
title: "ArgoCD 심화 보충 1: Source Hydrator vs Image Updater"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced supplement
---

## Source Hydrator
- manifest 전체 변환 (helm/kustomize → plain).
- hydrated brunch에 commit.
- 변환 결과 audit.

## Image Updater
- image tag만 자동 업데이트.
- manifest repo에 PR.

## 결합

둘 다 사용 가능:
- Image Updater가 tag 변경.
- Source Hydrator가 hydrated brunch에 변경 반영.

## 결정

- 변환 투명성 중요 → Hydrator.
- 단순 tag 자동 → Image Updater.
- 둘 다 → 결합.
