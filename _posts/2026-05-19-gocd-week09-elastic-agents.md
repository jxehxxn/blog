---
layout: post
title: "GoCD Week 9: Elastic Agents — Docker, Kubernetes"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd elastic-agent
---

## 학습 목표

- Elastic Agent의 의미.
- Docker / Kubernetes 플러그인.
- Profile + 자동 scaling.
- 비용 절감.

## 1. 비유 — "정직원 vs 일용직"

정직원(static agent): 항상 대기. 비싸지만 즉시 가용.

일용직(elastic agent): 작업 있을 때만 띄움. 끝나면 종료. 비용 절감.

## 2. Elastic Agent Profile

```yaml
elastic_profiles:
  - id: docker-jdk17
    plugin_id: cd.go.contrib.elasticagent.docker
    properties:
      Image: openjdk:17
      Command: |
        sh -c "$(curl -sSL https://gocd.org/agent-bootstrap.sh)"
      Environment: ["GO_SERVER_URL=https://gocd-server:8154/go"]
```

Job에서 사용:
```yaml
jobs:
  compile:
    elastic_profile_id: docker-jdk17
    tasks: [...]
```

## 3. Docker Plugin

Docker daemon 필요. 노드에서 container 띄움.

## 4. Kubernetes Plugin

K8s에 Pod로 agent:

```yaml
elastic_profiles:
  - id: k8s-build
    plugin_id: cd.go.contrib.elasticagent.kubernetes
    properties:
      Image: gocd/gocd-agent-alpine:v25
      Cluster_Profile: my-k8s
      Pod_Spec_Type: yaml
      Pod_Spec: |
        apiVersion: v1
        kind: Pod
        spec:
          containers:
            - name: agent
              resources:
                requests: { cpu: 500m, memory: 1Gi }
                limits: { cpu: 2, memory: 4Gi }
```

K8s가 Pod scheduling. 작업 끝나면 Pod 삭제.

## 5. Cluster Profile

```yaml
cluster_profiles:
  - id: my-k8s
    plugin_id: cd.go.contrib.elasticagent.kubernetes
    properties:
      kubeconfig: |
        ...
      go_server_url: https://gocd.mycorp.com/go
      auto_register_timeout: 10
```

## 6. 자동 scaling

GoCD가 pending job 보고 agent 띄움. 작업 끝나면 idle agent 자동 종료 (auto_register_timeout 후).

비유: 손님 오면 직원 호출, 가면 보냄.

## 7. 비용 절감

static agent 50대 = 항상 가동 비용.
elastic agent = 평균 5대 가동 → 90% 절감.

빅테크 표준: prod-critical만 static, 나머지 elastic.

## 8. 운영 함정 5선

1. **Pod resource limit 누락** → 한 job이 노드 점령.
2. **Image cache 미사용** → 매번 pull (느림).
3. **kubeconfig 노출**.
4. **auto_register_timeout 너무 짧음** → 시작도 안 한 agent 종료.
5. **Spot instance + 긴 작업** → 중간 종료.

## 9. 실습 과제

1. K8s plugin 설치.
2. cluster profile + elastic profile 설정.
3. Pipeline job에 elastic_profile_id 적용.
4. trigger → Pod 자동 생성 확인.
5. 작업 끝 → Pod 자동 삭제 확인.

## 10. 자가평가 퀴즈

### Q1. Elastic Agent 비유?
1. **일용직 (필요할 때 띄움)**
2. 정직원 3. UI 4. 무관

**정답: 1.**

### Q2. Plugin 2종?
1. **Docker / Kubernetes**
2. AWS only 3. UI 4. 무관

**정답: 1.**

### Q3. Pod 종료 시점?
1. **작업 완료 + auto_register_timeout 후**
2. 영원히 3. UI 4. 무관

**정답: 1.**

### Q4. 비용 절감 비율?
1. **70~90% (평균 사용량 기준)**
2. 5% 3. UI 4. 무관

**정답: 1.**

### Q5. 빅테크 패턴?
1. **prod-critical static, 나머지 elastic**
2. 모두 static 3. 모두 elastic 4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 10: Plugins + Secret Management].
