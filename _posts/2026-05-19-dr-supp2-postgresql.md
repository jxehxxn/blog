---
layout: post
title: "DR 보충 2: PostgreSQL DR 깊게"
date: 2026-05-19 00:00:00 +0900
categories: dr postgresql supplement
---

## WAL Archiving
S3에 WAL 지속 push. PITR 기반.

## pgBackRest
산업 표준 backup tool.

## CloudNativePG operator
K8s native PG, backup + PITR + DR 통합.

## 결론
DB는 별도 전문 도구. K8s만으로 부족.
