---
layout: post
title: "ArgoCD Perf 최종 자가평가"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance assessment
---

각 정답 1.

## Q1. 첫 병목? **controller**.
## Q2. 큰 app 분담? **sharding**.
## Q3. consistent hashing? **재할당 최소**.
## Q4. repo-server bound? **CPU**.
## Q5. shallow clone? **latency 절감**.
## Q6. Redis HA? **redis-ha sentinel**.
## Q7. Redis eviction? **LRU**.
## Q8. ApplicationSet 위험? **Matrix 폭발**.
## Q9. requeue 권장? **60s+**.
## Q10. Webhook 가치? **즉시 sync**.
## Q11. Webhook 부담? **push 폭주**.
## Q12. 5k app memory? **16~32GB**.
## Q13. OperationProcessors 1000+? **50**.
## Q14. APF 효과? **K8s API 우선순위**.
## Q15. etcd? **큰 자원 → disk**.
## Q16. reconcile p95? **30s**.
## Q17. queue depth 알람? **100**.
## Q18. drift 감지? **OutOfSync 비율**.
## Q19. kubectl exec pending? **K8s API throttle**.
## Q20. Sync queue 막힘? **statusProcessors / shard**.
## Q21. OOM? **limit + cache TTL**.
## Q22. Git timeout? **replica + mirror**.
## Q23. Diff 계산 비싼? **큰 app**.
## Q24. ServerSideApply? **optimization**.
## Q25. cache hit 효과? **render 0**.
## Q26. live cache TTL? **짧음**.
## Q27. benchmark 도구? **argocd CLI + Prometheus**.
## Q28. Intuit reference? **10k app**.
## Q29. fail-open? **ArgoCD 정지 시 cluster 영향 X**.
## Q30. capacity planning? **benchmark 기반**.
