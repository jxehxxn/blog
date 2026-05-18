---
layout: post
title: "IaC Week 4: Modules + Terragrunt — DRY at Scale"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- Module의 책임 분리 원리를 안다.
- Public registry 모듈 vs 자체 모듈을 구별한다.
- Terragrunt의 DRY 패턴을 안다.
- 빅테크의 module versioning 전략을 익힌다.

## 1. 비유 — "건축 자재"

집을 지을 때마다 못·기둥·창문을 처음부터 설계하지 않습니다. 표준 자재(모듈)를 골라 조립. 빅테크도 똑같이 **사내 표준 모듈**로 인프라를 조립합니다.

## 2. 모듈 기본

```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
```

사용:
```hcl
module "prod_vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
  azs    = ["a", "b", "c"]
}
```

source 종류:
- 로컬 (`./modules/vpc`)
- Git (`git::https://github.com/mycorp/tf-modules.git//vpc?ref=v1.2.0`)
- Terraform Registry (`hashicorp/vpc/aws`)
- S3, GCS

## 3. 모듈 디자인 5계명

1. **작게**: 한 책임 (VPC만, EKS만). 거대 모듈 금지.
2. **버전 고정**: `?ref=v1.2.0`. 절대 main 사용 금지 (변경에 휘말림).
3. **명시적 input/output**: 모든 var에 description + type. output도 마찬가지.
4. **기본값 합리적**: production-safe default.
5. **README + example**: 사용자가 5분 안에 이해 가능.

자세한 anti-pattern은 [보충 2: Module 디자인 패턴].

## 4. Public Registry 모듈

대표: `terraform-aws-modules/*`.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"
  ...
}
```

장점: 검증된 패턴, 빠른 시작.
단점: 회사 규정과 안 맞을 수 있음 (tag 정책, encryption default 등).

빅테크 표준: **public 모듈을 wrapping한 사내 모듈** 운영. 회사 정책을 default로 강제.

## 5. Terragrunt — Terraform의 한계 보완

Terraform 한계:
- 환경(dev/prod)마다 backend 설정 중복.
- 모듈 input의 공통값 중복.
- "이 모듈 → 저 모듈" 의존성 관리 어려움.

Terragrunt(`gruntwork-io/terragrunt`)가 thin wrapper로 보완.

### 디렉토리

```
live/
  prod/
    network/
      terragrunt.hcl
    eks/
      terragrunt.hcl
  dev/
    network/
      terragrunt.hcl
terragrunt.hcl     # 루트 공통
```

### 루트 terragrunt.hcl
```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "mycorp-tf-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-lock"
    encrypt        = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
}
EOF
}
```

### 자식 terragrunt.hcl
```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "git::https://github.com/mycorp/tf-modules.git//vpc?ref=v1.2.0"
}

inputs = {
  cidr = "10.0.0.0/16"
  azs  = ["a", "b", "c"]
}
```

이러면:
- backend 설정 자동 상속.
- provider 자동 생성.
- 환경별 디렉토리에서 그냥 `terragrunt apply`.

### dependency 명시
```hcl
dependency "vpc" {
  config_path = "../network"
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
}
```

자동 실행 순서 결정 (`terragrunt run-all apply` 시 network → eks 순서).

## 6. Module versioning 전략

### Semantic Versioning
`MAJOR.MINOR.PATCH`. breaking change는 MAJOR.

### Branch 정책
- `main`: 다음 release 후보.
- tag (`v1.2.0`): immutable.
- 사용자는 항상 tag 참조.

### Release process
1. PR 머지 → `main`.
2. `git tag v1.2.0 && git push --tags`.
3. CHANGELOG.md 업데이트.

빅테크 일부는 monorepo + 다양한 모듈 + 각자 tag (terraform-aws-modules 패턴).

## 7. Module 사용 빅테크 흐름

```
[ 사내 module repo ]
  └─ vpc, eks, rds, s3, iam, ... (각자 tag)
        ↓
[ live config repo ]
  ├─ prod/network → vpc 모듈
  ├─ prod/eks     → eks 모듈
  └─ ...
```

플랫폼팀은 모듈 관리, 앱팀은 live config만. 책임 분리.

## 8. 실습

```bash
# 1. 간단한 VPC 모듈 작성 (./modules/vpc)
# 2. main에서 module "vpc" {} 로 호출
# 3. version 고정해 git source로 변경
# 4. Terragrunt 설치 후 live/dev/network → vpc 모듈 호출
# 5. dependency로 eks가 vpc output 참조
```

## 9. 자가평가 퀴즈

### Q1. Module 디자인 5계명 중 가장 위험한 위반?
1. **version 미고정 (main branch 직접 참조)**
2. README 부재
3. 변수명
4. 무관

**정답: 1.**

### Q2. Public 모듈을 그대로 vs 래핑?
1. 그대로
2. **사내 정책(태그/암호화) 강제 래핑**
3. 무관
4. 사용 금지

**정답: 2.**

### Q3. Terragrunt의 가장 큰 가치?
1. 빠른 CLI
2. **backend/provider/dependency 중복 제거 + DRY**
3. UI
4. 무관

**정답: 2.**

### Q4. dependency 명시의 효과?
1. 무관
2. **run-all 실행 순서 자동 결정**
3. UI
4. 비용

**정답: 2.**

### Q5. monorepo 모듈 + 다양한 tag의 장점?
1. **한 곳에서 관리 + 모듈별 독립 버전**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 5: Terraform Workflow]에서 Atlantis/Spacelift/Env0 같은 GitOps CI를 다룹니다.
