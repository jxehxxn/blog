---
layout: post
title: "DR Week 3: Velero 기초"
date: 2026-05-19 00:00:00 +0900
categories: dr velero senior-series
---

## 설치

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws \
  --bucket mycorp-backup \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1
```

## Backup

```bash
velero backup create my-backup --include-namespaces payments
```

## Restore

```bash
velero restore create --from-backup my-backup
```

## Schedule

```bash
velero schedule create daily --schedule="0 2 * * *" --ttl 720h
```

## 자가평가
### Q1. backup 단위? **namespace/cluster**. 정답 1.
### Q2. schedule? **cron + TTL**. 정답 1.
### Q3. storage? **S3/GCS/Azure**. 정답 1.
### Q4. restore? **별도 cluster도 가능**. 정답 1.
### Q5. 빈도? **daily + critical은 hourly**. 정답 1.
