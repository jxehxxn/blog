---
layout: post
title: "Events 보충 4: Jenkins Trigger → Argo Events"
date: 2026-05-18 23:00:00 +0900
categories: argo-events supplement
---

## Jenkins Triggers
- SCM polling.
- Webhook.
- Cron.

## Argo Events 매핑
- SCM polling → GitHub source.
- Webhook → webhook source.
- Cron → calendar source.

## Migration
단계:
1. parallel 운영.
2. 한 job씩 sensor로.
3. Jenkins 제거.

## 결론
6개월 plan, 점진.
