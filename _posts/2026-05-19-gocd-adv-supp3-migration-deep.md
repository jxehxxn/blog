---
layout: post
title: "GoCD 심화 보충 3: Spinnaker/Bamboo/TeamCity/Concourse 마이그레이션"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd migration supplement
---

## Spinnaker → GoCD
- Spinnaker pipeline UI/JSON → yaml.
- canary/blue-green 패턴 직접 구현 (Spinnaker가 더 native).
- Cloud Driver는 GoCD task에서 cloud CLI.

## Bamboo (Atlassian) → GoCD
- 비슷한 stage/job 구조.
- yaml 변환 도구 있음 (사내 자체 작성 필요).

## TeamCity (JetBrains) → GoCD
- TeamCity의 build chains → GoCD pipeline dependency.
- 자세한 reporting은 GoCD가 약함 (TeamCity 강점).

## Concourse → GoCD
- Concourse가 pipeline DSL이지만 모델 다름.
- materials = resources.
- jobs → stages/jobs.

## 공통 plan
- 6개월 점진.
- 1~3 pilot.
- 5~10 표준 정착.
- 50+ scale.
- sunset.

## 결론
도구 변경 = 조직 변경. 6개월.
