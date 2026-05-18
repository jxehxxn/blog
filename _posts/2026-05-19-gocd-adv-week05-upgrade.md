---
layout: post
title: "GoCD 심화 Week 5: Upgrade Strategy (Zero Downtime)"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd advanced upgrade senior
---

## 학습 목표
- GoCD upgrade 절차.
- Plugin compatibility.
- Rolling upgrade (HA).
- Rollback.

## 1. Release cadence
ThoughtWorks가 분기 minor + 필요 시 patch.

## 2. Upgrade 종류
- minor: 보통 호환.
- major: breaking change 가능.

## 3. 절차

### Pre
1. CHANGELOG 검토.
2. Plugin 호환 확인.
3. Staging cluster에서 test.
4. Full backup.

### Apt/RPM
```bash
apt update && apt install go-server go-agent
systemctl restart go-server
```

### Docker
새 image tag로 restart.

### Helm
```bash
helm upgrade gocd gocd/gocd --version 1.X.0
```

## 4. HA Rolling
- Standby 먼저 upgrade.
- LB로 traffic 전환.
- Primary upgrade.
- LB 복귀.

zero downtime.

## 5. Plugin Compatibility

Server 버전 ↔ plugin 버전. 매트릭스 확인.

## 6. Rollback

apt/yum downgrade 또는 Docker 이전 tag.

DB 변경이 있었다면 rollback 어려움 (단방향).

## 7. 운영 함정
1. Plugin 호환 안 봄.
2. Backup 없이 upgrade.
3. Major upgrade 한 번에.
4. Production만 upgrade (staging skip).

## 8. 자가평가
### Q1. Release cadence? **분기 minor**. 정답 1.
### Q2. HA upgrade? **Standby 먼저**. 정답 1.
### Q3. Plugin 호환? **matrix 확인**. 정답 1.
### Q4. Rollback 어려운 경우? **DB schema 변경**. 정답 1.
### Q5. major upgrade? **점진, staging 먼저**. 정답 1.
