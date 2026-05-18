---
layout: post
title: "GoCD 심화 최종 자가평가 — 30문항 (정답+해설)"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd advanced assessment
---

각 정답 1.

## 인프라
### Q1. HA 모델? **Active-Passive**.
### Q2. 외부 DB 권장? **PostgreSQL 16+**.
### Q3. Artifact 공유? **NFS/S3**.
### Q4. embedded H2? **production 금지**.
### Q5. Disk purge? **purgeStartDiskSpace + purgeUptoDiskSpace**.
### Q6. JVM GC? **G1GC**.
### Q7. Polling vs Webhook? **Webhook 우선**.
### Q8. UI p95 SLO? **2s**.
### Q9. Backup API? **POST /api/backups**.
### Q10. Restore drill? **분기 1회**.

## Upgrade/Plugin
### Q11. Release cadence? **분기 minor**.
### Q12. HA upgrade? **Standby 먼저**.
### Q13. Major rollback 어려운 이유? **DB schema 변경**.
### Q14. SSO plugin 종류? **LDAP/SAML/OAuth**.
### Q15. ChatOps? **Slack slash command**.
### Q16. Plugin = ? **jar file**.
### Q17. Plugin 위치? **plugins/external/**.
### Q18. Task plugin request? **get-view + execute**.
### Q19. Secret plugin? **{{SECRET:...}} resolve**.
### Q20. Elastic Agent 4 method? **assign/create/cancel/status**.

## API/Governance
### Q21. Auth 표준? **Personal Access Token**.
### Q22. Accept header? **application/vnd.go.cd+json**.
### Q23. Trigger API? **POST /api/pipelines/.../schedule**.
### Q24. Pipeline Template 가치? **표준화**.
### Q25. Governance? **template + lint + audit**.
### Q26. Monorepo? **material filter + 여러 pipeline**.
### Q27. Blue-Green? **두 환경 + 전환**.
### Q28. Canary monitor? **metric query stage**.
### Q29. SOC 2 CC8.1? **PR + audit + IaC**.
### Q30. SOX? **7년 audit + SoD**.

## 채점
- 28~30: 빅테크 GoCD senior.
- 24~27: 미드.
- 18~23: 주니어 → 보강 영역 학습.
- ~17: 기본 코스 재복습.

수고하셨습니다. 이제 진짜 시니어급 GoCD 운영 가능.
