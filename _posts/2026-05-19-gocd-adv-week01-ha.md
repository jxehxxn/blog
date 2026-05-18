---
layout: post
title: "GoCD 심화 Week 1: Server HA + 외부 DB"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd advanced ha senior
---

## 학습 목표
- GoCD Server HA 아키텍처.
- 외부 DB (PostgreSQL/MySQL) 설정.
- Load balancer + session 공유.

## 1. 비유 — "공장 본부 2동"
한 본부(server) 화재 시 다른 본부가 즉시 인수. 책상(DB)은 별도 안전 금고에.

## 2. HA 아키텍처

```
[ LB (NGINX/HAProxy) ]
       ↓
[ Server A (active) ]  [ Server B (standby) ]
       ↓                       ↓
       [ External PostgreSQL HA ]
       [ Shared NFS/S3 (artifacts) ]
```

Active-Passive (기본). Active 죽으면 Standby가 즉시 활성.

## 3. 외부 DB

기본 H2 (embedded). production = 외부 PostgreSQL.

```properties
# config/db.properties
db.driverClass=org.postgresql.Driver
db.url=jdbc:postgresql://pg.mycorp.com:5432/gocd
db.user=gocd
db.password=...
```

PostgreSQL 16+ 권장. HA replication.

## 4. Shared Artifact Storage

Server 둘이 같은 artifact 접근 필수:
- NFS (전통).
- S3 (cloud) — plugin.
- Azure Blob.

## 5. Failover

수동 (DNS 변경) 또는 자동 (LB health check).

빅테크 표준: LB health check + Pacemaker.

## 6. Limitation

GoCD 자체는 active-active 미지원. active-passive만.

## 7. 운영 함정 5선
1. embedded DB 그대로 → 데이터 손실.
2. Artifact storage 분리 안 함.
3. session sticky 누락.
4. Standby 갱신 누락 (config out of sync).
5. Failover 미시험.

## 8. 자가평가
### Q1. 외부 DB 권장? **PostgreSQL 16+**. 정답 1.
### Q2. HA 모델? **Active-Passive**. 정답 1.
### Q3. Artifact storage? **NFS/S3 등 공유**. 정답 1.
### Q4. embedded H2 위험? **데이터 손실**. 정답 1.
### Q5. failover? **수동 DNS 또는 LB health check**. 정답 1.
