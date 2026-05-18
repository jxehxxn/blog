---
layout: post
title: "Events 보충 2: Sensor Pattern Catalog"
date: 2026-05-18 23:00:00 +0900
categories: argo-events supplement
---

## 1. Single source → single trigger
가장 단순. webhook → workflow.

## 2. Multi-source AND
두 event 모두 와야 trigger.

## 3. Multi-trigger
한 event → 여러 trigger (workflow + slack + lambda).

## 4. Chained
trigger A 결과 → trigger B. 여러 sensor 연결.

## 5. Time-window
시간 안에 N개 event 모이면 trigger.

## 6. Idempotent
중복 event 처리 안 함 (idempotency key).

## 결론
패턴 카탈로그가 운영의 핵심.
