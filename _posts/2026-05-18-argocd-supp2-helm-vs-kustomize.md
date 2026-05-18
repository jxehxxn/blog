---
layout: post
title: "ArgoCD 보충 2: Helm vs Kustomize — 정량 비교와 혼합 패턴"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes supplement
---

5주차에서 짧게 결정한 Helm vs Kustomize를 정량적으로 비교합니다.

## 1. 본질 차이

- **Helm**: Go template engine으로 manifest를 생성.
- **Kustomize**: 기존 manifest에 patch overlay 적용.

근본적으로 Helm은 **생성기**, Kustomize는 **편집기**.

## 2. 정량 비교

| 항목 | Helm | Kustomize |
|------|------|-----------|
| 학습 곡선 | 가파름 (template 문법) | 완만 (YAML 그대로) |
| 외부 패키지 통합 | 표준 (chart ecosystem) | 어려움 |
| 환경별 변형 | values 분기 (복잡) | overlay (자연스러움) |
| 의존성 관리 | requirements.yaml | components |
| Rollback | history 내장 | git revert |
| 동적 값 | template function 강력 | replacement/var 제한적 |
| Big org governance | chart museum, version pin | overlay 표준화 가능 |
| Conditional logic | `{{- if ... }}` 강력 | 어려움 |

## 3. 사용 시점 결정

```
외부 third-party (postgres, redis, kafka) ? 
  → Helm (해당 chart가 ecosystem에)
사내 마이크로서비스 ? 
  → Kustomize base/overlay
GitOps 환경 분기(dev/stage/prod) ?
  → Kustomize overlay
복잡한 conditional ?
  → Helm
```

## 4. 혼합 패턴 1: 분리

- 외부 chart는 Helm Application으로.
- 사내 manifest는 Kustomize Application으로.
- 같은 ArgoCD가 둘 다 자동 인식.

## 5. 혼합 패턴 2: Helm chart에 Kustomize overlay

ArgoCD plugin으로 `helm template` 출력을 다시 Kustomize로 후가공.

```yaml
plugin:
  name: helm-kustomize
```

CMP 스크립트:

```bash
helm template . | kustomize build -k - 
```

빅테크 케이스: vendor chart는 그대로 받지만 사내 정책(security context, resource limit)을 overlay로 강제.

## 6. 정량 사례 — manifest 변경 시 영향 범위

같은 변경 "이미지 태그 update"를 두 도구에서:

### Helm
- `values-prod.yaml`의 `image.tag` 한 줄 수정.
- `helm template` 결과 N개 manifest 모두 새 tag.

### Kustomize
- `overlays/prod/kustomization.yaml`의 `images.newTag` 한 줄.
- 결과 동일.

둘 다 한 줄. 비등.

같은 변경 "특정 환경만 resource limit 강화":

### Helm
- values 분기 + template `{{- if .Values.prod }}`.
- chart 자체에 분기 로직 침투. dev 변경이 prod에 영향 가능.

### Kustomize
- overlays/prod에 patch 한 개.
- base는 무관, 영향 격리.

이 경우 Kustomize가 우세.

## 7. 보안 관점

- Helm: 외부 chart의 무결성. Helm chart 서명, OCI registry checksum.
- Kustomize: 자체 manifest 무결성. Git signed commit.

빅테크 표준: chart는 사내 mirror에서만, Kustomize base는 protected branch에서만.

## 8. 빅테크 사례 — Spotify, Adobe

Spotify는 외부 dependency는 Helm, 사내 마이크로서비스는 Kustomize. Adobe는 거의 모든 사내 서비스를 Kustomize, vendor만 Helm.

## 9. 결론

"하나만 골라" 라는 질문 자체가 잘못. 빅테크는 두 도구를 함께 쓰며 강점만 취합니다. ArgoCD가 둘을 자연스럽게 통합하기에 분리 비용도 작습니다.
