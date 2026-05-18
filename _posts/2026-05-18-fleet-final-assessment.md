---
layout: post
title: "Fleet 최종 자가평가 — 30문항"
date: 2026-05-18 23:30:00 +0900
categories: fleet assessment
---

각 정답 1.

## Q1. CAPI 책임? **K8s cluster 자체 CRD**.
## Q2. Management vs workload? **Mgmt가 다른 cluster 생성**.
## Q3. Providers 3종? **Bootstrap/ControlPlane/Infra**.
## Q4. CAPA? **AWS provider**.
## Q5. KCP replicas 표준? **3**.
## Q6. MD vs MS vs Machine? **계층 (Deployment 패턴)**.
## Q7. MachinePool? **cloud ASG wrap**.
## Q8. OCM 책임? **multi-cluster 운영**.
## Q9. Hub/Klusterlet? **중앙/agent**.
## Q10. ManifestWork? **manifest 분배**.
## Q11. Placement? **배포 cluster 선택**.
## Q12. PlacementDecision? **선택 결과**.
## Q13. ArgoCD ApplicationSet 중앙? **중앙 ArgoCD 장애 전체 영향**.
## Q14. ArgoCD per cluster? **운영 N배 but 격리**.
## Q15. CAPI + ArgoCD bootstrap? **cluster 생성 → 자체 ArgoCD**.
## Q16. CAPI vs OCM? **생성 vs 운영**.
## Q17. Upgrade 전략? **canary → wave → 전체**.
## Q18. management cluster 손실? **모든 workload orphan**.
## Q19. Cluster delete? **CAPI가 cloud 정리**.
## Q20. kubent? **deprecated API 검사**.
## Q21. Multi-cluster metric? **Thanos/Mimir 통합**.
## Q22. EKS variant? **AWSManagedControlPlane**.
## Q23. KubeFed? **deprecated**.
## Q24. Rancher Fleet? **Rancher의 GitOps**.
## Q25. Cluster 생성 후 ArgoCD bootstrap? **self-install pattern**.
## Q26. CAPI controller 종류? **cluster-api/bootstrap/cp/infra**.
## Q27. webhook? **admission validation**.
## Q28. cluster naming? **CR name → cloud 자원**.
## Q29. OCM ApplicationSet generator? **ManagedCluster → Application**.
## Q30. tooling? **kubent + CAPI upgrade plan**.

다음: Backup/DR.
