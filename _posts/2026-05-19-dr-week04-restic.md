---
layout: post
title: "DR Week 4: Restic / Kopia Volume Backup"
date: 2026-05-19 00:00:00 +0900
categories: dr velero restic senior-series
---

## CSI snapshot 한계
- 일부 CSI 만 지원.
- block 만.
- snapshot이 cloud 자체 dependent.

## Restic / Kopia
file-system level backup. PVC 내용을 S3로 복사.

Velero integration:
```bash
velero backup create my-backup --default-volumes-to-fs-backup
```

## Trade-off
- Restic: 느림, 호환 좋음.
- CSI snapshot: 빠름, cloud 특화.

## 자가평가
### Q1. Restic? **file-level backup**. 정답 1.
### Q2. CSI snapshot 한계? **일부 CSI만**. 정답 1.
### Q3. Velero integration? **--default-volumes-to-fs-backup**. 정답 1.
### Q4. trade-off? **속도 vs 호환**. 정답 1.
### Q5. cross-cluster restore? **Restic이 더 자유**. 정답 1.
