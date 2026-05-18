---
layout: post
title: "Workflow Week 7: Tekton Triggers, Resolvers, Chains"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- Tekton Triggers로 외부 event → PipelineRun.
- Resolvers로 원격 Task/Pipeline 참조.
- Chains로 공급망 서명·증명.

## 1. Tekton Triggers

### 3 컴포넌트

- **EventListener**: HTTP server (Service로 노출).
- **TriggerBinding**: HTTP payload → parameter 추출.
- **TriggerTemplate**: parameter → PipelineRun manifest.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata: { name: github-listener }
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push
      interceptors:
        - ref: { name: github }
          params:
            - { name: secretRef, value: { secretName: gh-secret, secretKey: secretToken } }
            - { name: eventTypes, value: [push] }
      bindings:
        - ref: github-binding
      template:
        ref: build-template
```

### Binding
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata: { name: github-binding }
spec:
  params:
    - { name: repo-url, value: $(body.repository.clone_url) }
    - { name: revision, value: $(body.head_commit.id) }
```

### Template
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata: { name: build-template }
spec:
  params:
    - name: repo-url
    - name: revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata: { generateName: build- }
      spec:
        pipelineRef: { name: build-and-test }
        params:
          - { name: repo-url, value: $(tt.params.repo-url) }
          - { name: revision, value: $(tt.params.revision) }
```

EventListener의 Service URL을 GitHub webhook으로.

## 2. Resolvers (v0.40+)

원격 Task/Pipeline을 cluster에 설치 없이 참조.

### Git resolver
```yaml
taskRef:
  resolver: git
  params:
    - { name: url, value: https://github.com/tektoncd/catalog.git }
    - { name: revision, value: main }
    - { name: pathInRepo, value: task/git-clone/0.9/git-clone.yaml }
```

### Bundles resolver (OCI)
```yaml
taskRef:
  resolver: bundles
  params:
    - { name: bundle, value: gcr.io/tekton-releases/catalog/upstream/git-clone:0.9 }
    - { name: name, value: git-clone }
    - { name: kind, value: task }
```

### Hub resolver
```yaml
taskRef:
  resolver: hub
  params:
    - { name: name, value: git-clone }
    - { name: version, value: "0.9" }
```

빅테크 패턴: 자체 Git/OCI repo의 Task를 resolver로. 사내 표준 강제.

## 3. Tekton Chains — 공급망 보안

빌드 → 자동 (1) 산출물 메타데이터 기록 (provenance), (2) 서명 (cosign/in-toto), (3) attestation 저장.

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
```

각 TaskRun/PipelineRun 완료 시 Chains controller가 자동:
1. 산출물 (image) hash 추출.
2. SLSA provenance 생성.
3. cosign으로 서명.
4. transparency log (Rekor)에 적재.

SLSA Level 3까지 자동.

자세한 supply chain은 [보충 2: Tekton Chains 깊게] + Tier 1 Course 7.

## 4. PipelineRun Spec — Advanced

### Pod template
```yaml
spec:
  taskRunTemplate:
    podTemplate:
      nodeSelector: { workload: ci }
      tolerations: [{ key: ci, operator: Exists }]
      securityContext: { runAsNonRoot: true }
```

### Parameter validation
```yaml
spec:
  params:
    - name: env
      type: string
      enum: [dev, stage, prod]
```

v1 (stable) 부터 enum 지원.

## 5. Tekton 운영 함정 5선

1. EventListener Service 외부 노출 시 secret 검증 누락 → 외부 trigger.
2. Resolver 사용 시 외부 의존성 → 외부 장애에 PipelineRun fail.
3. PVC workspace + 다중 node → mount conflict.
4. Chains 미설정 → 공급망 evidence 없음.
5. TaskRun TTL 미설정 → 자원 누적.

## 6. 빅테크 사례 — Google Cloud Build

Google Cloud Build가 Tekton 엔진. 사내 사용 + 외부 SaaS 제공. Chains로 SLSA 자동.

## 7. 실습

```bash
# 1. Tekton Triggers 설치
# 2. GitHub EventListener + secret 검증
# 3. Resolver로 catalog Task 참조
# 4. Chains 설치 후 PipelineRun 결과의 cosign signature 확인
```

## 8. 자가평가 퀴즈

### Q1. Triggers 3 컴포넌트?
1. **EventListener, TriggerBinding, TriggerTemplate**
2. Source/Bus/Sensor
3. UI
4. 무관

**정답: 1.**

### Q2. Resolver의 가치?
1. **Task/Pipeline을 cluster 설치 없이 원격 참조**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. Chains의 가치?
1. **자동 서명·증명 + SLSA 준수**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. EventListener secret 검증 누락 위험?
1. **외부 누구나 trigger**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. enum 활용 가치?
1. **잘못된 parameter 값 차단**
2. UI
3. 빠름
4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 8: Argo Workflows vs Tekton]에서 의사결정 기준을 정리합니다.
