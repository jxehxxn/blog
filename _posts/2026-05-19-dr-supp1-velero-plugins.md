---
layout: post
title: "DR 보충 1: Velero Plugins"
date: 2026-05-19 00:00:00 +0900
categories: dr velero supplement
---

## BackupItemAction / RestoreItemAction
사용자 정의 hook. 특정 자원 backup/restore 변형.

## 예
- Secret 마스킹.
- Cross-namespace mapping.

## 결론
Velero 확장의 핵심.
