---
layout: post
title: "ArgoCD 심화 Week 3: CMP (Config Management Plugin) 개발"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced platform senior-series
---

## 학습 목표

- CMP 아키텍처.
- sidecar 패턴 vs 옛 method.
- 직접 plugin 작성.

## 1. CMP

Helm/Kustomize/Jsonnet 외 도구 통합. cdk8s, ytt, Tanka, kpt 등.

## 2. v2.4+ sidecar 패턴

repo-server에 sidecar 추가. plugin이 sidecar에서 실행.

```yaml
# argocd-cm config
configManagementPlugins: |
  - name: ytt
    init:
      command: ["sh", "-c"]
      args: ["ytt --version"]
    generate:
      command: ["sh", "-c"]
      args: ["ytt -f ."]
```

옛 in-process 방식은 deprecated.

## 3. ConfigManagementPlugin CR (sidecar)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata: { name: my-cmp }
spec:
  version: v1.0
  init:
    command: ["sh"]
    args: ["init.sh"]
  generate:
    command: ["sh"]
    args: ["generate.sh"]
  discover:
    fileName: "*.yaml"
```

repo-server sidecar에서 실행.

## 4. Sidecar Image

```dockerfile
FROM alpine:3.18
RUN apk add curl bash
COPY init.sh generate.sh /
USER 999
ENTRYPOINT ["/var/run/argocd/argocd-cmp-server"]
```

argocd-cmp-server binary를 base.

## 5. Application 사용

```yaml
spec:
  source:
    plugin:
      name: my-cmp
      parameters:
        - name: env
          value: prod
```

parameter는 환경변수 또는 stdin.

## 6. discover

repo에 자동 매칭 (어느 path가 이 plugin 사용할지):
```yaml
discover:
  fileName: "kustomization.yaml"
  find:
    glob: "**/Tanka.jsonnet"
    command: ["sh", "-c", "find . -name Tanka.jsonnet"]
```

## 7. Best Practice

- init은 dep 설치, generate는 실제 변환.
- output은 stdout (yaml 또는 json).
- error는 stderr.
- timeout 합리적 (5분).
- multi-stage Docker image로 크기 줄임.

## 8. 운영 함정

1. sidecar OOM (large repo).
2. plugin 버전 mismatch.
3. ConfigManagementPlugin CR 필수 (v2.6+).
4. parameter 누락.
5. discover 충돌 (여러 plugin 매칭).

## 9. 자가평가

### Q1. CMP 가치?
1. **임의 manifest 도구 통합** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. sidecar vs in-process?
1. **sidecar 표준 (v2.4+), in-process deprecated** 2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q3. discover?
1. **어느 repo path가 이 plugin 사용할지 자동 매칭** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Best practice — init?
1. **dep 설치 (generate에서 install 금지)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. output 형식?
1. **stdout YAML/JSON** 2. file 3. UI 4. 무관

**정답: 1.**

## 10. 다음

[Week 4: ApplicationSet plugin generator].
