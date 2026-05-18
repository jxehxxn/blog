---
layout: post
title: "DR Week 7: Stateful Workload DR (DB)"
date: 2026-05-19 00:00:00 +0900
categories: dr database senior-series
---

## PostgreSQL
- WAL streaming (synchronous/async).
- PITR (point-in-time recovery).
- Patroni operator.

## MySQL
- async replication.
- Group Replication.

## NoSQL
- Cassandra: multi-DC replication 표준.
- Mongo: replica set + DR set.

## DB Operator
Zalando postgres-operator, CloudNativePG.

## 자가평가
### Q1. PostgreSQL DR? **WAL streaming + PITR**. 정답 1.
### Q2. Patroni? **PG HA operator**. 정답 1.
### Q3. Cassandra DR? **multi-DC replication**. 정답 1.
### Q4. CloudNativePG? **PG operator**. 정답 1.
### Q5. sync vs async? **latency vs RPO trade-off**. 정답 1.
