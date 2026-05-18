---
layout: post
title: "GoCD 보충 2: Value Stream Map 깊게"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd vsm supplement
---

7주차 VSM을 시각화·운영 수준으로.

## 1. 기원
VSM은 도요타 TPS (Toyota Production System)의 lean manufacturing 도구. CD에 적용된 첫 도구 중 하나가 GoCD.

## 2. 무엇을 시각화하나
- **단계별 lead time**: 각 stage 평균 시간.
- **wait time**: stage 사이 기다림.
- **분기/병합**: fan-out, fan-in.
- **manual gate**: 사람 결재 위치.

## 3. 측정 + 개선

비유: 공장 흐름도에 빨간 점(병목) 표시 → 개선 대상.

GoCD VSM에서:
- 가장 긴 stage → 최적화.
- 가장 긴 wait → 자동화 또는 approval 줄임.
- 자주 fail하는 stage → reliability 작업.

## 4. End-to-end Lead Time

commit ~ prod 도달 시간. CD 책에서 가장 중요 지표.

빅테크 목표:
- Elite: < 1 hour.
- High: < 1 day.
- Medium: 1d ~ 1w.

DORA metrics 표준.

## 5. Cycle Time

한 변경의 발의 ~ 완료 시간. lead time과 구분.

## 6. VSM을 dashboard로

GoCD UI 외에:
- Grafana plugin.
- 사내 Backstage 통합.

## 7. 결론

VSM = CD 운영의 지도. 시니어가 dashboard에 매일 보고 개선 대상 식별.
