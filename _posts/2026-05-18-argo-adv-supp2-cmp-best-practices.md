---
layout: post
title: "ArgoCD 심화 보충 2: CMP Best Practices"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced supplement
---

## 1. init vs generate

- init: dep 설치 (1회).
- generate: 매 sync 마다.

generate에서 install 금지 (느림).

## 2. Output 표준

stdout YAML 또는 JSON. error는 stderr.

## 3. Timeout

repo-server 기본 90초. plugin 자체 timeout 더 짧게.

## 4. 메모리

큰 manifest는 sidecar OOM 위험. resource limit.

## 5. 보안

plugin이 외부 git/registry 접근 → CA + token 관리.

## 6. 버전 관리

plugin 이미지 tag 고정. semver.

## 7. 테스트

unit test (script) + integration test (sample manifest).

## 8. 결론

CMP는 ArgoCD 확장의 핵심. 운영 규율이 가치.
