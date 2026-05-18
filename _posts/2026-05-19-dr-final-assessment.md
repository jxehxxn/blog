---
layout: post
title: "DR 최종 자가평가"
date: 2026-05-19 00:00:00 +0900
categories: dr assessment
---

각 정답 1.

## Q1. RTO? **복구 허용 시간**.
## Q2. RPO? **데이터 손실 허용**.
## Q3. Tier 0? **Active-active, RPO 0**.
## Q4. etcd snapshot 빈도? **매시간**.
## Q5. Velero backup 단위? **namespace/cluster**.
## Q6. Velero schedule? **cron + TTL**.
## Q7. Restic? **file-level**.
## Q8. CSI snapshot 한계? **일부 CSI만**.
## Q9. VolumeSnapshot? **PVC PIT**.
## Q10. Restore PVC? **dataSource**.
## Q11. Active-Active? **RTO 0**.
## Q12. Warm vs cold? **운영 중 vs 정지**.
## Q13. PG DR? **WAL streaming**.
## Q14. PITR? **point-in-time recovery**.
## Q15. Cassandra DR? **multi-DC replication**.
## Q16. GitOps DR 가치? **빠른 재배포**.
## Q17. Stateful DR? **PV + DB 별도**.
## Q18. Secret DR? **Vault HA**.
## Q19. 복원 순서? **bootstrap→secret→CRD→ns→stateful→stateless**.
## Q20. Tabletop? **회의실 시나리오**.
## Q21. Full failover 빈도? **연 1회**.
## Q22. 측정? **실제 vs 목표**.
## Q23. RPO 0 요구? **sync replication**.
## Q24. Aurora Global? **RPO < 1초**.
## Q25. SOC 2 A1.2? **backup 정기**.
## Q26. HIPAA? **contingency plan**.
## Q27. 증빙? **drill 결과**.
## Q28. Chaos Mesh? **K8s native chaos**.
## Q29. cross-cloud? **극히 어려움**.
## Q30. CloudNativePG? **PG operator + DR**.
