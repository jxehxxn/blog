---
layout: post
title: "Interview Week 7: Design — Logging System"
date: 2026-05-19 08:00:00 +0900
categories: interview design senior-series
---

## Prompt
"Design a logging system for 1000 services."

## 답
1. **Collection**: Fluent Bit DaemonSet.
2. **Buffer**: Kafka.
3. **Storage**: Loki (cost) 또는 Elasticsearch (full-text).
4. **UI**: Grafana / Kibana.
5. **Retention**: tiered (hot 30d, cold 1y).
6. **Cost**: label cardinality 관리.
7. **Scale**: per region cluster.

## 자가평가
### Q1. Collection? **Fluent Bit DaemonSet**. 정답 1.
### Q2. Buffer? **Kafka**. 정답 1.
### Q3. Loki vs ELK? **cost vs full-text**. 정답 1.
### Q4. tiered retention? **hot/cold**. 정답 1.
### Q5. cardinality? **cost 핵심**. 정답 1.
