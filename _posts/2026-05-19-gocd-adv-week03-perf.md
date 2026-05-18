---
layout: post
title: "GoCD 심화 Week 3: Performance Tuning"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd advanced performance senior
---

## 학습 목표
- 병목 위치.
- JVM tuning.
- DB index.
- 큰 cluster 운영.

## 1. 병목
1. Server JVM heap (큰 pipeline 수).
2. DB query (pipeline history).
3. Material polling (수천 git repo).
4. Artifact I/O.
5. Agent communication.

## 2. JVM Tuning

```bash
GO_SERVER_SYSTEM_PROPERTIES="-Xms4g -Xmx8g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled"
```

GC pause 짧게. heap 큰 환경 (수천 pipeline).

## 3. DB Index

PostgreSQL slow query log → 자주 미인덱싱.
자주 index 필요:
- pipeline_id
- counter
- last_modified

## 4. Material Polling

기본 60초. 큰 환경에서 부담:
```properties
go.server.materials.poll.interval.minutes=2
```

또는 webhook 우선.

## 5. Agent Throughput

agent 메모리 / CPU 제한.
큰 빌드는 dedicated agent. 작은 빌드는 elastic.

## 6. SLO
- UI response p95 < 2s.
- pipeline trigger latency < 30s.
- agent assignment < 30s.

## 7. 자가평가
### Q1. JVM GC 권장? **G1GC**. 정답 1.
### Q2. heap? **수천 pipeline 4~8GB**. 정답 1.
### Q3. polling 권장? **webhook 우선**. 정답 1.
### Q4. DB? **PostgreSQL + index 튜닝**. 정답 1.
### Q5. UI p95? **2s**. 정답 1.
