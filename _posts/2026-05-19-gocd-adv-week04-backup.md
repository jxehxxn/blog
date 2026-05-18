---
layout: post
title: "GoCD 심화 Week 4: Backup / Restore / DR"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd advanced dr senior
---

## 학습 목표
- Server 데이터 분류.
- 자동 backup.
- Restore drill.
- Multi-region DR.

## 1. 데이터 분류

GoCD server는 4종 데이터:
- **config**: `config/cruise-config.xml` (pipeline 정의).
- **DB**: PostgreSQL (history, run).
- **artifacts**: `godata/artifacts/`.
- **plugins**: `plugins/external/`.

## 2. Backup API

```bash
curl -X POST -u admin \
  http://gocd/go/api/backups \
  -H "Accept: application/vnd.go.cd+json"
```

server가 4종 모두 묶음. `server-backup-YYYYMMDD.zip` 생성.

S3 자동 push:
```properties
backup.upload.s3.bucket=mycorp-gocd-backup
backup.upload.s3.region=us-east-1
```

## 3. Schedule

```yaml
backup_schedule:
  cron: "0 0 2 * * ?"   # daily 2am
```

## 4. Restore

새 server에:
1. zip extract.
2. config, DB, plugins 복원.
3. artifacts NFS/S3 attach.
4. server start.

분기 1회 drill 필수.

## 5. Multi-region DR

- Primary region: active server.
- Standby region: 최근 backup + idle server.
- Failover 30분~1시간.

## 6. RTO/RPO 목표
- RTO: 1h (Standby 활성화).
- RPO: 24h (daily backup) or 1h (continuous DB replication).

## 7. 운영 함정
1. Backup 안 함.
2. Restore drill 미시행.
3. Artifact 큰 cluster에서 backup 실패 (timeout).
4. Standby 갱신 누락.

## 8. 자가평가
### Q1. 4종 데이터? **config/DB/artifacts/plugins**. 정답 1.
### Q2. Backup API? **/api/backups POST**. 정답 1.
### Q3. drill? **분기 1회**. 정답 1.
### Q4. RTO 목표? **1h**. 정답 1.
### Q5. continuous RPO? **DB replication**. 정답 1.
