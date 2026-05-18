---
layout: post
title: "ArgoCD Week 5: 리포지토리 도구 — Helm, Kustomize, Jsonnet, CMP"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 학습 목표

- ArgoCD가 지원하는 4가지 manifest 생성 방식을 안다.
- Helm vs Kustomize의 본질적 차이와 트레이드오프를 이해한다.
- Config Management Plugin(CMP)으로 임의의 도구 통합 방법을 익힌다.
- 빅테크의 "Helm + Kustomize 혼합" 패턴을 적용한다.

## 1. 4가지 매니페스트 생성 방식

ArgoCD repo-server는 다음을 자동 인식·렌더:

1. **Raw YAML**: `path` 안의 모든 yaml을 그대로.
2. **Helm**: `Chart.yaml`이 있으면 `helm template`.
3. **Kustomize**: `kustomization.yaml`이 있으면 `kustomize build`.
4. **Jsonnet**: `*.jsonnet` 파일 인식.
5. **CMP (Config Management Plugin)**: 그 외 — Tanka, ytt, cdk8s 등.

## 2. Helm 통합

```yaml
spec:
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: 14.0.0
    helm:
      releaseName: my-postgres
      valueFiles:
        - values-prod.yaml
      values: |
        image:
          tag: 16.1.0
      parameters:
        - name: replicaCount
          value: "3"
```

빅테크 패턴: `valueFiles`로 env별 분리 + `values`/`parameters`로 동적 오버라이드.

## 3. Kustomize 통합

```yaml
spec:
  source:
    repoURL: https://github.com/mycorp/manifests.git
    targetRevision: HEAD
    path: overlays/prod
    kustomize:
      namePrefix: prod-
      images:
        - "mycorp/webapp=mycorp/webapp:v2.3.4"
      commonLabels:
        env: prod
```

`overlays/prod`에서 base를 reference하고 patch를 얹는 표준 패턴.

## 4. Helm vs Kustomize — 한 줄 결정

- **Helm**: 외부 제공 패키지(써드파티 차트) 통합에 좋음. template engine.
- **Kustomize**: 사내 manifest의 환경별 변형에 좋음. patch overlay.

자세한 정량 비교는 [보충 2: Helm vs Kustomize] 포스트.

### 빅테크 혼합 패턴

```
base/                        # Kustomize base
  - deployment.yaml
  - service.yaml
overlays/
  prod/                      # Kustomize overlay
    - kustomization.yaml
    - patch-prod.yaml
charts/
  postgres/                  # Helm chart 별도 Application
```

Application 2개를 만든다:
- `webapp-prod`: path=`overlays/prod` (Kustomize)
- `postgres-prod`: chart=`postgresql` (Helm)

각 도구의 강점만 취합니다.

### Helm + Kustomize 동시 사용 (post-render)

```yaml
kustomize:
  ...
helm:
  ...
```

는 직접 결합되지 않습니다. 대신 ArgoCD는 `helm template`의 결과를 다시 Kustomize로 후가공하는 패턴을 plugin으로 지원.

## 5. Jsonnet 통합

```yaml
spec:
  source:
    repoURL: https://github.com/mycorp/manifests.git
    targetRevision: HEAD
    path: jsonnet
    directory:
      jsonnet:
        extVars:
          - name: env
            value: prod
```

Jsonnet은 표현력이 높지만 학습 곡선이 가팔라 작은 팀에서 부담. 빅테크 중에서도 Grafana Labs(Tanka)·일부 SRE팀이 즐겨 씁니다.

## 6. CMP — 임의의 도구 통합

ArgoCD repo-server에 sidecar로 plugin 등록.

```yaml
# argocd-cm
configManagementPlugins: |
  - name: cdk8s
    generate:
      command: ["sh", "-c"]
      args: ["cdk8s synth && cat dist/*.yaml"]
```

사용:

```yaml
spec:
  source:
    plugin:
      name: cdk8s
```

빅테크에서 cdk8s(TypeScript), ytt(Carvel), Pulumi 등 도입 시 사용.

## 7. Multi-source Application (v2.6+)

여러 source를 한 Application으로:

```yaml
spec:
  sources:
    - repoURL: https://charts.bitnami.com/bitnami
      chart: postgresql
      helm:
        valueFiles:
          - $values/values-prod.yaml
    - repoURL: https://github.com/mycorp/manifests.git
      ref: values
```

`$values/`로 외부 values 파일을 다른 git에서 참조. 차트는 vendor, 값은 사내 관리.

## 8. 실전 함정 5선

1. **Helm `lookup` 함수**: ArgoCD는 dry-run으로 렌더하므로 `lookup`이 비어 옴.
2. **Kustomize `vars` (deprecated)**: replacement로 대체.
3. **외부 chart의 보안 검토 누락**: 사내 mirror에 통합 필수 (5주차 trufflehog 강의와 동일 패턴).
4. **너무 깊은 overlay**: base → mid → prod 3단 이상은 추적이 지옥.
5. **Plugin 의존성 폭발**: CMP 너무 많으면 repo-server 의존성 관리가 어려움.

## 9. 빅테크 사례 — Spotify의 Helm + Kustomize 혼합

Spotify는 외부 dependency(Postgres, Redis)는 Helm chart로, 사내 마이크로서비스는 Kustomize base/overlay로 관리한다고 발표했습니다. 두 도구를 모두 ArgoCD가 자동 인식해 일관된 운영이 가능.

## 10. 실습 과제

1. 동일 nginx Deployment를 (a) Raw YAML (b) Helm chart (c) Kustomize base/overlay 세 방식으로 작성.
2. 각각을 Application으로 등록 후 sync.
3. UI에서 렌더 결과 비교.
4. valueFiles로 dev/prod 두 환경 분리.
5. CMP로 간단한 sh script 기반 generator 작성.

## 11. 자가평가 퀴즈

### Q1. 외부 third-party 패키지 통합에 가장 적합한 도구는?
1. Helm
2. Kustomize
3. Raw YAML
4. Jsonnet

**정답: 1.**

### Q2. 사내 manifest의 env별 변형에 가장 권장되는 도구는?
1. Kustomize overlay
2. Helm template 분기
3. Jsonnet
4. Raw YAML

**정답: 1.** (단, 외부 chart + sub-values는 Helm.)

### Q3. Helm `lookup` 함수가 ArgoCD에서 비어 오는 이유는?
1. ArgoCD는 dry-run 렌더라 클러스터 조회를 안 함
2. 권한 부족
3. 버전 호환성
4. 의미 없음

**정답: 1.**

### Q4. multi-source Application의 활용은?
1. chart는 vendor, values는 사내 git에서 가져오기
2. 두 source를 동시에 적용
3. backup
4. 의미 없음

**정답: 1.**

### Q5. CMP는 언제 사용?
1. ArgoCD가 기본 지원하지 않는 manifest 도구(cdk8s, ytt 등) 통합
2. 모든 Application
3. 보안 강화
4. CI 빌드 가속

**정답: 1.**

## 12. 다음 주차

[Week 6: 멀티클러스터 / 멀티테넌트]에서는 hub-spoke vs decentralized 아키텍처를 비교하고, 1개 ArgoCD가 N개 클러스터를 관리하는 운영 노하우를 정리합니다.
