---
layout: post
title: "DR Week 5: Volume Snapshot CSI 깊게"
date: 2026-05-19 00:00:00 +0900
categories: dr csi snapshot senior-series
---

## VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: pg-snap }
spec:
  volumeSnapshotClassName: ebs
  source: { persistentVolumeClaimName: pg-data }
```

## Restore

snapshot → 새 PVC:
```yaml
spec:
  dataSource:
    name: pg-snap
    kind: VolumeSnapshot
```

## Cross-region copy
EBS snapshot copy. CSI driver별 다름.

## 자가평가
### Q1. VolumeSnapshot? **PVC PIT**. 정답 1.
### Q2. restore? **dataSource로 새 PVC**. 정답 1.
### Q3. cross-region? **EBS copy**. 정답 1.
### Q4. 한계? **block CSI만 일반적**. 정답 1.
### Q5. Velero 통합? **자동**. 정답 1.
