---
layout: post
title: "GoCD 보충 3: K8s Elastic Agent 설정 가이드"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd elastic-agent k8s supplement
---

9주차 K8s elastic agent 운영.

## 1. 사전

- K8s cluster (kind/EKS/GKE).
- gocd-server가 cluster에서 접근 가능.

## 2. Plugin 설치

```bash
# UI Admin → Plugins → Upload
# kubernetes-elastic-agent.jar
```

## 3. Cluster Profile

```yaml
cluster_profiles:
  - id: prod-k8s
    plugin_id: cd.go.contrib.elasticagent.kubernetes
    properties:
      go_server_url: https://gocd.mycorp.com/go
      auto_register_timeout: 10
      pending_pods_count: 10
      kubernetes_cluster_url: https://k8s-api.mycorp.com
      kubernetes_cluster_ca_cert: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      kubernetes_cluster_namespace: gocd-agents
      security_token: <K8s SA token>
```

## 4. Elastic Profile

```yaml
elastic_profiles:
  - id: jdk17-agent
    plugin_id: cd.go.contrib.elasticagent.kubernetes
    cluster_profile_id: prod-k8s
    properties:
      Image: gocd/gocd-agent-alpine:v25
      Pod_Spec_Type: yaml
      Pod_Spec: |
        apiVersion: v1
        kind: Pod
        spec:
          serviceAccountName: gocd-agent
          containers:
            - name: agent
              image: gocd/gocd-agent-alpine:v25
              resources:
                requests: { cpu: 500m, memory: 1Gi }
                limits: { cpu: 2, memory: 4Gi }
              env:
                - name: JDK_VERSION
                  value: "17"
```

## 5. RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: gocd-agent, namespace: gocd-agents }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch", "create", "delete"]
```

## 6. 사용

```yaml
jobs:
  compile:
    elastic_profile_id: jdk17-agent
    tasks:
      - exec:
          command: bash
          arguments: ["-c", "mvn clean install"]
```

## 7. 운영 모니터링

```bash
kubectl -n gocd-agents get pods -w
```

Pending → Running → Completed → 자동 삭제.

## 8. 결론

K8s elastic agent로 GoCD가 cloud-native 시대에 맞춤.
