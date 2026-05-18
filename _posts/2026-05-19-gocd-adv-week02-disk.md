---
layout: post
title: "GoCD 심화 Week 2: Disk + Artifact Retention"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd advanced senior
---

## 학습 목표
- Artifact 저장 모델.
- Retention policy.
- Disk monitoring + cleanup.

## 1. 위치
`server.artifacts.dir` (기본 `godata/artifacts`).

각 pipeline run마다 폴더. 수천 pipeline × 수만 run = TB+ 가능.

## 2. Retention 정책

```xml
<server artifactsdir="artifacts">
  <purgeSettings>
    <purgeStartDiskSpace>20</purgeStartDiskSpace>  <!-- GB 남으면 purge 시작 -->
    <purgeUptoDiskSpace>30</purgeUptoDiskSpace>     <!-- GB까지 purge -->
  </purgeSettings>
</server>
```

또는 pipeline 별:
```yaml
clean_working_directory: true
```

## 3. Manual Cleanup

```bash
curl -X DELETE -u admin \
  http://gocd/go/api/admin/pipelines/my-pipeline/instances/<counter>
```

특정 run 삭제.

## 4. S3 Plugin Retention

S3로 보내면 lifecycle policy로 90일 후 삭제.

## 5. Monitoring

```promql
# disk usage
node_filesystem_avail_bytes{mountpoint="/godata"}
```

알람: < 50GB.

## 6. 운영 함정
1. purge 미설정 → disk full → server 정지.
2. 짧은 retention → debug 불가.
3. NFS network slow → fetch artifact 지연.

## 7. 자가평가
### Q1. artifact dir? **godata/artifacts**. 정답 1.
### Q2. purge 설정? **purgeStartDiskSpace + purgeUptoDiskSpace**. 정답 1.
### Q3. S3 lifecycle? **자동 삭제**. 정답 1.
### Q4. monitor? **disk avail bytes**. 정답 1.
### Q5. disk full? **server 정지**. 정답 1.
