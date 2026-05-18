---
layout: post
title: "ArgoCD Week 8: 시크릿 GitOps — Vault, SealedSecrets, SOPS, Image Updater"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes security secrets
---

## 학습 목표

- "Git에 평문 시크릿 두기"의 본질 문제를 안다.
- External Secrets Operator + Vault 패턴을 구현한다.
- SealedSecrets, SOPS의 트레이드오프를 비교한다.
- ArgoCD Image Updater로 이미지 태그 GitOps 자동화를 적용한다.

## 1. 본질 문제

GitOps는 desired state를 Git에 둡니다. 그런데 시크릿(API 키, DB 패스워드)을 Git에 평문으로 두면? **누가 리포에 접근하면 다 보임**. Public 리포는 말할 것도 없고 private 리포도 위험합니다 (TruffleHog 강의 Week 1 참고).

해법은 모두 같은 원칙: **Git에는 암호화된 참조만, 평문은 런타임에 풀어 클러스터로**.

## 2. 옵션 비교

| 옵션 | 평문 보관 위치 | 운영 | 권장 |
|------|---------------|------|------|
| External Secrets Operator + Vault | Vault | 가장 강력, 운영 복잡 | 대규모 |
| Bitnami SealedSecrets | 클러스터 (controller가 복호화) | 단순 | 중소규모 |
| SOPS | 키 관리 서비스(KMS/PGP) | git-encrypt 흐름 단순 | 소규모/CI 중심 |
| 평문 (금지) | Git | — | 절대 X |

## 3. External Secrets + Vault 패턴

### 흐름

```
[ Vault ] ←─ secret 저장
   ↑
[ External Secrets Operator ]
   ↓ (주기적 fetch)
[ K8s Secret 생성 ]
   ↓ (mount)
[ Pod ]
```

Git에는 평문이 없습니다. ESO가 Vault에서 가져와 K8s Secret을 만듭니다.

### ExternalSecret CR

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-secret
  data:
    - secretKey: username
      remoteRef:
        key: secret/data/payments/db
        property: username
    - secretKey: password
      remoteRef:
        key: secret/data/payments/db
        property: password
```

빅테크 패턴: ESO + Vault dynamic secret(매번 새 DB 자격증명 발급) → 누출되어도 자동 만료.

## 4. SealedSecrets

Bitnami SealedSecrets controller가 클러스터에 설치되어 있고, 그 controller만 가진 개인키로 복호화.

```bash
# 일반 Secret 작성
echo -n 'p@ssw0rd' | kubectl create secret generic db --dry-run=client \
  --from-file=password=/dev/stdin -o yaml > db-secret.yaml

# Sealed로 변환
kubeseal -f db-secret.yaml -w db-sealed.yaml --controller-namespace kube-system
```

`db-sealed.yaml`은 평문이 아닌 암호문이라 Git에 둬도 안전. 클러스터의 controller만 풀 수 있음.

장단점:
- + 별도 secret manager 없이 시작 가능.
- − 키 회전이 번거롭고 (controller 키 변경 시 모든 sealed 재암호화 필요), 클러스터 간 portability 제한.

## 5. SOPS

Mozilla SOPS로 YAML/JSON 파일을 부분 암호화. 키는 AWS KMS / GCP KMS / age / PGP 중 선택.

```bash
sops -e -i db-secret.yaml
# 파일 안 password 필드만 암호화, 나머지는 평문
```

ArgoCD에서 사용 시 plugin(예: argocd-vault-plugin, sops plugin)으로 복호화.

## 6. ArgoCD Image Updater — 이미지 태그 GitOps

매번 새 이미지가 빌드될 때 Git에 tag를 업데이트해야 하는데, CI가 직접 git commit하면 GitOps 일관성이 깨질 수 있습니다. ArgoCD Image Updater가 ArgoCD 안에서 처리.

```yaml
# Application annotation
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: webapp=mycorp/webapp
    argocd-image-updater.argoproj.io/webapp.update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

설정:
- semver / latest / digest / regex / alphabetical update strategy.
- write-back: argocd 자체 commit 또는 git PR.

빅테크 사용 패턴: PR 모드로만 운영, 사람이 리뷰 후 머지 → 자동 sync.

## 7. 실전 함정 5선

1. **ESO refresh가 너무 짧음**: Vault rate limit 침범.
2. **SealedSecrets 키 회전 누락**: controller 키 침해 시 모든 secret 위험.
3. **SOPS 키 분실**: 복호화 불가 → secret 영구 손실. 키 백업 절차 필수.
4. **Image Updater write-back direct**: PR 리뷰 단계 없이 main에 직접 commit → 변경 추적 약화.
5. **Vault 토큰 만료**: ESO/ArgoCD가 Vault 인증 토큰 회전 자동화 필요.

## 8. 빅테크 사례 — Adobe의 ESO + Vault

Adobe는 ESO + HashiCorp Vault를 표준화해 사내 거의 모든 시크릿을 Vault 단일 source-of-truth로 두고 있습니다. dynamic secret으로 DB 자격증명 자동 회전, ESO로 K8s Secret 동기화. ArgoCD는 K8s Secret 자체를 GitOps에서 제외(ignoreDifferences).

## 9. 실습 과제

1. Vault dev 모드 + ESO 설치.
2. ExternalSecret CR로 mock DB credential 동기화.
3. SealedSecrets로 동일 시크릿을 Git에 안전하게 둠.
4. SOPS로 같은 시크릿을 KMS 키로 부분 암호화.
5. Image Updater로 자기 리포의 이미지 태그 자동 PR 생성.

## 10. 자가평가 퀴즈

### Q1. Git에 평문 시크릿을 두면 안 되는 핵심 이유는?
1. 누가 리포 접근 시 즉시 노출, 회수 어려움
2. UI 표시 문제
3. 빌드 느림
4. 비용

**정답: 1.**

### Q2. SealedSecrets의 핵심 약점은?
1. controller 키 회전 시 모든 sealed 재암호화 필요
2. 단순함
3. UI 부재
4. 무관

**정답: 1.**

### Q3. ESO + Vault dynamic secret의 장점은?
1. 누출돼도 자동 만료, 회수 부담 감소
2. UI 색상
3. 비용 절감
4. 무관

**정답: 1.**

### Q4. Image Updater write-back 모드로 권장되는 것은?
1. PR 리뷰 후 머지
2. main에 직접 commit
3. 무관
4. 의미 없음

**정답: 1.**

### Q5. SOPS의 핵심 위험은?
1. 키 분실 → 복호화 영구 불가
2. 단순함
3. UI
4. 비용

**정답: 1.**

## 11. 다음 주차

[Week 9: Progressive Delivery]에서는 Argo Rollouts로 Canary / Blue-Green / Analysis-based 배포를 다룹니다.
