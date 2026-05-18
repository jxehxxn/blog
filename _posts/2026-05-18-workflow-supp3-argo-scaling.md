---
layout: post
title: "Workflow 보충 3: Argo Workflows Controller 성능 튜닝"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series supplement
---

대규모(수천 workflow 동시) Argo Workflows 운영 깊게.

## 1. 병목 위치

1. controller CPU (reconcile loop).
2. etcd I/O (수천 Pod CRD update).
3. archive DB (PostgreSQL).
4. UI server.

## 2. Controller 튜닝

### Workers
```yaml
workflow-controller-configmap:
  data:
    workflowDefaults: |
      ...
    parallelism: 100      # 전체 워크플로 동시 실행 한도
```

CLI:
```bash
workflow-controller --workflow-workers 64 --pod-workers 64
```

## 3. Pod GC

```yaml
spec:
  podGC:
    strategy: OnPodSuccess   # 성공 Pod 즉시 삭제
```

큰 workflow는 수천 Pod 누적 → etcd 부하. GC 필수.

## 4. Archive

완료 workflow를 etcd에서 SQL (PostgreSQL/MySQL)로 이전.

```yaml
config: |
  persistence:
    postgresql:
      host: pg.argo.svc
      port: 5432
      database: argo
      tableName: argo_workflows
      userNameSecret: { name: argo-pg-creds, key: user }
      passwordSecret: { name: argo-pg-creds, key: password }
    archiveTTL: 30d
```

ArchiveTTL: 30일 후 PG에서도 삭제.

UI는 PG 조회해 옛 workflow 보여줌.

## 5. Workflow TTL

```yaml
spec:
  ttlStrategy:
    secondsAfterCompletion: 86400   # 1일
    secondsAfterSuccess: 86400
    secondsAfterFailure: 604800     # 7일
```

성공은 빨리, 실패는 디버깅을 위해 길게.

## 6. Pod Resource Defaults

```yaml
workflowDefaults: |
  spec:
    podSpecPatch: |
      containers:
        - name: main
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits: { cpu: 2, memory: 4Gi }
```

요청 안 한 workflow도 default. cluster 안정성.

## 7. UI 분리

UI가 controller와 같은 process면 부하 경합. v3.5+ argo-server 별도 deployment.

```bash
kubectl -n argo scale deploy argo-server --replicas=3
```

## 8. Metric 모니터링

```promql
# 현재 active workflow
argo_workflows_count{status="Running"}

# reconcile latency
histogram_quantile(0.95, rate(argo_workflows_workflow_condition_seconds_bucket[5m]))

# operation duration
histogram_quantile(0.95, rate(argo_workflows_operation_duration_seconds_bucket[5m]))

# error rate
sum(rate(argo_workflows_error_count[5m])) by (cause)
```

알람: reconcile p95 > 30초, error rate spike.

## 9. Sharding (per-namespace)

여러 controller가 namespace 분담:
```bash
workflow-controller --namespaced --managed-namespace ns-A
```

각 controller가 자기 namespace만. blast radius 분리.

## 10. 결론

기본 설정으로는 100~500 workflow가 한계. 위 튜닝으로 1만+ 도달. 빅테크는 sharding + PG archive + GC + TTL을 모두 적용합니다.
