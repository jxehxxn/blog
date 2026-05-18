---
layout: post
title: "Workflow Week 9: CI 패턴 — Build, Test, Scan, Sign, Deploy"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- 빅테크 CI의 표준 5단계 패턴.
- Kaniko/Buildah로 daemonless build.
- Trivy로 vulnerability scan.
- Cosign으로 image 서명.
- ArgoCD trigger로 deploy.

## 1. 표준 5단계

```
[Fetch/Lint] → [Test] → [Build] → [Scan + Sign] → [Deploy]
```

각 단계가 격리된 Pod. 실패 시 다음 단계 차단.

## 2. Fetch + Lint

- git-clone (Tekton catalog 또는 git artifact).
- golangci-lint, eslint, flake8 등.

```yaml
# Tekton Task
spec:
  steps:
    - name: lint
      image: golangci/golangci-lint:v1.55
      workingDir: $(workspaces.source.path)
      script: |
        golangci-lint run ./...
```

## 3. Test

```yaml
- name: test
  image: golang:1.21
  workingDir: $(workspaces.source.path)
  script: |
    go test -race -coverprofile=coverage.out ./...
    go tool cover -func=coverage.out
```

결과를 Sonarqube/Codecov로 업로드.

## 4. Build — Daemonless

Docker daemon 없이 컨테이너 빌드. 보안 강화.

### Kaniko
```yaml
- name: build
  image: gcr.io/kaniko-project/executor:latest
  args:
    - --dockerfile=Dockerfile
    - --context=$(workspaces.source.path)
    - --destination=$(params.image):$(params.tag)
    - --cache=true
    - --cache-repo=$(params.image)-cache
  volumeMounts:
    - { name: docker-config, mountPath: /kaniko/.docker }
```

### Buildah
Red Hat 진영. OCI 표준 준수.

### Argo Workflows 패턴
```yaml
- name: build
  inputs:
    artifacts:
      - { name: src, path: /workspace }
  container:
    image: gcr.io/kaniko-project/executor:latest
    args: ["--dockerfile=Dockerfile", "--context=/workspace", "--destination={{workflow.parameters.image}}:{{workflow.parameters.tag}}"]
```

## 5. Scan — Trivy

```yaml
- name: scan
  image: aquasec/trivy:latest
  script: |
    trivy image --exit-code 1 --severity HIGH,CRITICAL \
      $(params.image):$(params.tag)
```

CRITICAL/HIGH 발견 시 fail. 결과를 SARIF/JSON으로 SIEM 적재.

## 6. Sign — Cosign

```yaml
- name: sign
  image: gcr.io/projectsigstore/cosign:v2.2.0
  env:
    - { name: COSIGN_EXPERIMENTAL, value: "1" }
  script: |
    cosign sign --yes --key=k8s://signing/cosign-key \
      $(params.image):$(params.tag)
```

또는 keyless (Sigstore + Rekor):
```bash
cosign sign --yes <image>
# OIDC token (GitHub Actions, GitLab CI 등)으로 인증
```

자세한 supply chain은 Tier 1 Course 7.

## 7. Deploy — ArgoCD Trigger

CI는 image tag만 결정. 실제 deploy는 GitOps.

방법 1: Image Updater (ArgoCD 강의 Week 8).
방법 2: CI가 manifest repo에 PR 생성.

```yaml
- name: update-manifest
  script: |
    git clone https://github.com/mycorp/manifests.git
    cd manifests
    yq e '.spec.template.spec.containers[0].image = "$(params.image):$(params.tag)"' \
      -i apps/myapp/deployment.yaml
    git commit -am "bump myapp to $(params.tag)"
    git push
```

PR로 manifest 변경 → ArgoCD 자동 sync.

## 8. Argo Workflows로 같은 패턴

```yaml
spec:
  entrypoint: ci
  templates:
    - name: ci
      dag:
        tasks:
          - name: lint
            template: lint
          - name: test
            template: test
            dependencies: [lint]
          - name: build
            template: build
            dependencies: [test]
          - name: scan
            template: scan
            dependencies: [build]
          - name: sign
            template: sign
            dependencies: [scan]
          - name: deploy
            template: update-manifest
            dependencies: [sign]
```

같은 5 단계, Argo로.

## 9. Cache 전략

- Image layer cache: Kaniko의 `--cache`, BuildKit의 export-cache.
- Dependency cache: go mod, npm, pip → workspace 또는 volume.
- Test cache: go test -count=1 막지 말 것.

## 10. 운영 함정 5선

1. **Docker-in-Docker (DinD)** 사용 → privileged Pod 필요, 보안 위험. Kaniko/Buildah 권장.
2. Cache 미사용 → 빌드 시간 폭증.
3. Trivy 임계 너무 낮음 → 모든 빌드 fail.
4. Cosign key 관리 부실 → 서명 의미 없음.
5. CI에서 직접 `kubectl apply` → GitOps 어김.

## 11. 빅테크 사례 — Shopify

Shopify는 모든 internal image cosign 서명 + admission webhook에서 검증. unsigned image는 cluster 진입 불가.

## 12. 실습

```bash
# 1. Tekton Pipeline: fetch → lint → test → build (Kaniko) → scan (Trivy) → sign (cosign) → update manifest
# 2. PipelineRun 실행 후 ArgoCD가 자동 sync
# 3. 일부러 vulnerability 있는 이미지 → scan fail 확인
# 4. cosign verify 로 서명 검증
```

## 13. 자가평가 퀴즈

### Q1. Daemonless 빌드 도구?
1. **Kaniko, Buildah**
2. docker
3. UI
4. 무관

**정답: 1.**

### Q2. Trivy의 가치?
1. **vulnerability scan**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. Cosign keyless 의 가치?
1. **OIDC 기반 서명 → 정적 key 관리 부담 제거**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. CI가 manifest PR 만드는 이유?
1. **GitOps 원칙 — ArgoCD가 sync, CI는 직접 apply 안 함**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. DinD 의 위험?
1. **privileged Pod 필요 → 보안 위험**
2. 빠름
3. UI
4. 무관

**정답: 1.**

## 14. 다음 주차

[Week 10: GitOps 통합]에서 Workflow → ArgoCD 완전 자동화를 다룹니다.
