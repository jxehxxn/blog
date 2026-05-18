---
layout: post
title: "IaC Week 5: Terraform Workflow — Atlantis, Spacelift, Env0로 GitOps화"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- 로컬 apply의 위험을 안다.
- Atlantis로 PR 기반 워크플로를 구축한다.
- Spacelift/Env0 같은 SaaS 옵션의 트레이드오프를 비교한다.
- PR comment 기반 plan/apply 자동화를 설계한다.

## 1. 비유 — "현장 시공 vs 도면 승인 후 시공"

엔지니어 개인 노트북에서 `terraform apply` = 현장 시공 직접. 위험: 비인가 변경, 검토 누락, 사고 발생 시 책임 추적 불가.

PR → CI plan → 리뷰 → merge → CI apply = "도면 승인 후 시공". 모든 변경이 PR 기록으로 audit 됨.

## 2. 로컬 apply의 7가지 위험

1. **비인가 변경**: 한 사람이 prod 자원 임의로 변경 가능.
2. **리뷰 부재**: 잘못된 plan을 검토 없이 적용.
3. **state lock 경쟁**: 다중 사용자 동시 작업 충돌.
4. **자격증명 분산**: prod 키가 모든 노트북에.
5. **버전 차이**: 사람마다 Terraform 버전 다름.
6. **재현성 부재**: "내 컴퓨터에서는 됐는데" 발생.
7. **Audit 부재**: 누가 언제 무엇을 변경했는지 추적 어려움.

## 3. Atlantis — OSS의 표준

GitHub/GitLab/Bitbucket PR에 plan을 자동 댓글로 게시, apply는 comment 명령으로.

### 흐름

```
[ 개발자 PR 생성 ]
     ↓
[ Atlantis webhook 받음 ]
     ↓ terraform plan
[ PR에 plan 결과 댓글 ]
     ↓ 리뷰
[ 리뷰어 "@atlantis apply" 코멘트 ]
     ↓ Atlantis가 apply
[ PR merge ]
```

### 설치 (Helm)
```bash
helm install atlantis runatlantis/atlantis \
  --set github.user=atlantis-bot \
  --set github.token=$GH_TOKEN \
  --set github.secret=$WEBHOOK_SECRET \
  --set orgAllowlist=github.com/mycorp/*
```

GitHub repo webhook URL: `https://atlantis.mycorp.com/events`.

### atlantis.yaml (repo 안)
```yaml
version: 3
projects:
  - dir: live/prod/network
    workspace: default
    autoplan:
      when_modified: ["*.tf", "*.tfvars"]
      enabled: true
    apply_requirements: [mergeable, approved]
  - dir: live/prod/eks
    workspace: default
    autoplan: { enabled: true }
```

apply 요구사항: PR 머지 가능 + 승인 받음. 안전.

### 권한 분리
Atlantis가 가진 AWS 권한이 핵심. 모든 사용자 노트북에 prod 권한 줄 필요 없음 — Atlantis만.

## 4. Spacelift / Env0 — SaaS 옵션

| 항목 | Atlantis | Spacelift | Env0 |
|------|----------|-----------|------|
| 호스팅 | 자체 | SaaS | SaaS |
| UI | 약함 | 강함 | 강함 |
| 정책 (OPA) | 외부 통합 | 내장 | 내장 |
| Drift 자동 감지 | 외부 | 내장 | 내장 |
| Multi-IaC (Pulumi/Crossplane) | X | O | O |
| 비용 | 무료 (OSS) | 유료 (조직 규모별) | 유료 |
| 빅테크 채택 | 보통 | 증가 | 증가 |

Spacelift는 "stack" 추상화 + run policy + drift detection을 한 화면에. 학습/도입 비용 적은 편.

## 5. GitHub Actions 자체 워크플로

Atlantis 없이 순수 Actions로도 가능:

```yaml
name: terraform
on:
  pull_request: { paths: ["**.tf"] }
  push: { branches: [main] }

permissions:
  contents: read
  pull-requests: write
  id-token: write   # OIDC for AWS

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/github-tf
          aws-region: us-east-1
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -out=tfplan -no-color | tee plan.txt
      - uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan.txt', 'utf8');
            github.rest.issues.createComment({
              ...context.repo,
              issue_number: context.issue.number,
              body: '```\n' + plan + '\n```',
            });
      - if: github.ref == 'refs/heads/main'
        run: terraform apply tfplan
```

빅테크에서 Actions OIDC + IAM role assume이 표준 — 정적 access key 제거.

## 6. Policy as Code 통합

`terraform plan -out=tfplan` 후 plan JSON을 OPA/Conftest로 검사.

```bash
terraform show -json tfplan > plan.json
conftest test plan.json --policy policies/
```

예: "production tag 없는 자원 거부", "큰 인스턴스 타입 거부", "S3 public access 거부". Tier 1 Course 6에서 깊게.

## 7. Drift Detection

Atlantis Pro / Spacelift는 주기적 plan으로 drift 감지. 코드와 현실이 어긋나면 알람.

OSS만으로: cron으로 `terraform plan -detailed-exitcode` 실행. exit 2 = drift, 알람.

## 8. 운영 함정 5선

1. **Atlantis 권한 과다**: 모든 자원에 admin → 침해 시 blast.
2. **apply_requirements 누락**: 리뷰 없이 apply.
3. **OIDC 미사용**: 정적 access key가 CI에 떠도는 위험.
4. **plan 결과 미저장**: audit evidence 부족.
5. **drift detection 부재**: 콘솔 직접 변경이 누적.

## 9. 빅테크 사례 — Netflix Atlantis

Netflix는 자체 Atlantis fork 운영. 수십 명이 매일 동시 PR. apply는 항상 Atlantis에서만, 노트북에서 prod 권한 0. audit log SIEM 적재.

## 10. 실습

```bash
# 1. Atlantis Helm 설치 (또는 docker run)
# 2. GitHub repo webhook 설정
# 3. PR 만들어 atlantis plan 자동 코멘트 확인
# 4. atlantis apply 명령
# 5. Spacelift 무료 trial로 같은 flow 비교
# 6. OPA로 policy 1개 추가 (S3 public 거부)
```

## 11. 자가평가 퀴즈

### Q1. 로컬 apply의 가장 큰 위험?
1. **비인가 변경 + audit 부재**
2. 빌드 느림
3. UI
4. 무관

**정답: 1.**

### Q2. Atlantis의 핵심 가치?
1. **PR 기반 plan/apply + 권한 집중 관리**
2. 더 빠른 plan
3. UI
4. 무관

**정답: 1.**

### Q3. OIDC + IAM role assume의 가치?
1. **정적 access key 제거 → 자격증명 누출 위험 차단**
2. 빠름
3. 무관
4. UI

**정답: 1.**

### Q4. apply_requirements의 효과?
1. **리뷰 + mergeable 조건 만족해야 apply — 안전 게이트**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q5. Drift detection의 필요?
1. **코드와 현실 불일치 자동 감지 → 사일런트 변경 차단**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 6: Crossplane 소개]에서 K8s가 universal control plane이 되는 발상을 다룹니다.
