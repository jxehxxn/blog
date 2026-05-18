---
layout: post
title: "DR Week 2: etcd Snapshot"
date: 2026-05-19 00:00:00 +0900
categories: dr etcd senior-series
---

## Snapshot

```bash
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
```

매 시간 cron + S3.

## Restore

```bash
etcdctl snapshot restore <file> --data-dir=/var/lib/etcd-new
```

API server 멈추고 manifest 수정.

## Verification
분기 drill.

## 자가평가
### Q1. 빈도? **매시간**. 정답 1.
### Q2. 저장? **S3 cross-region**. 정답 1.
### Q3. restore? **새 data-dir + manifest 수정**. 정답 1.
### Q4. drill? **분기 1회**. 정답 1.
### Q5. 손실 위험? **백업 검증 안 함**. 정답 1.
