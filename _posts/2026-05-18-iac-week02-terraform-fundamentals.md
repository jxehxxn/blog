---
layout: post
title: "IaC Week 2: Terraform 기초 — Provider, Resource, init/plan/apply 한 줄도 빠짐없이"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- Terraform 설치부터 첫 apply까지 30분에 완료한다.
- HCL 문법 핵심을 안다.
- `init / plan / apply / destroy` 4 단계 lifecycle을 외운다.
- Provider, Resource, Data source, Output, Variable을 구별한다.

## 1. 비유 — "건축 설계 4단계"

1. **init**: 도구함 준비 (망치/대패 챙기기).
2. **plan**: "도면대로 지으면 무엇이 어떻게 바뀌나?" 사전 시뮬레이션.
3. **apply**: 실제 시공.
4. **destroy**: 철거.

## 2. 설치

```bash
# macOS
brew install terraform

# Linux
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

terraform version
# Terraform v1.7+
```

OpenTofu (Terraform fork after license change) 도 같은 명령으로 사용. 빅테크 일부는 OpenTofu로 이주.

## 3. 첫 코드 — main.tf

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_s3_bucket" "demo" {
  bucket = "hoon-demo-bucket-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}

output "bucket_name" {
  value = aws_s3_bucket.demo.bucket
}
```

## 4. Lifecycle 4단계

### (1) init
```bash
terraform init
```
- `required_providers` 다운로드 → `.terraform/providers/`.
- backend 초기화 (state 파일 위치 결정).
- 모듈 다운로드.

### (2) plan
```bash
terraform plan -out=tfplan
```
- 현재 state와 desired(코드) 비교.
- `+ create`, `~ update`, `- destroy` 표시.
- 사람이 검토할 텍스트 출력. **운영에서는 plan 결과를 반드시 PR에 첨부**.

### (3) apply
```bash
terraform apply tfplan
```
- plan에서 결정된 변경을 클라우드 API로 실행.
- state 파일 갱신.

### (4) destroy
```bash
terraform destroy
```
- state 안 모든 리소스 제거. 위험.

빅테크 운영 팁: production에 destroy 금지 정책 + 임시 환경만 destroy 허용.

## 5. HCL 핵심 문법

### Variable
```hcl
variable "region" {
  type        = string
  default     = "ap-northeast-2"
  description = "AWS region"
}
```

사용: `provider "aws" { region = var.region }`

### Output
```hcl
output "url" {
  value     = aws_s3_bucket.demo.website_endpoint
  sensitive = false
}
```

다른 모듈에서 참조 또는 사람이 결과 확인.

### Data Source — 기존 자원 조회
```hcl
data "aws_vpc" "default" {
  default = true
}

resource "aws_subnet" "example" {
  vpc_id = data.aws_vpc.default.id
  cidr_block = "172.31.99.0/24"
}
```

기존 VPC를 만들지 않고 조회만.

### Locals — 임시 계산
```hcl
locals {
  common_tags = {
    Environment = "prod"
    Owner       = "platform-team"
  }
}

resource "aws_instance" "web" {
  tags = local.common_tags
}
```

### Conditionals & Loops
```hcl
resource "aws_instance" "web" {
  count = var.enable ? 1 : 0
}

resource "aws_security_group" "web" {
  for_each = toset(["80", "443"])
  ingress { from_port = tonumber(each.key), ... }
}
```

`count` vs `for_each`:
- `count`: 인덱스 기반. 중간 요소 삭제 시 뒤 인덱스 전부 재생성.
- `for_each`: 키 기반. 안전한 추가/삭제.

**대부분의 운영에서 for_each 권장**.

## 6. Provider — "어떤 클라우드와 대화하는가"

```hcl
provider "aws"     { region = "ap-northeast-2" }
provider "google"  { project = "my-project", region = "asia-northeast3" }
provider "azurerm" { features {} }
provider "kubernetes" { config_path = "~/.kube/config" }
provider "helm"      {}
provider "github"    { token = var.gh_token }
```

3,000+ provider 존재. 사내 도구도 자체 provider 작성 가능 (Go).

## 7. 의존성 — 명시적 vs 암시적

암시적 (참조):
```hcl
resource "aws_subnet" "a" {
  vpc_id = aws_vpc.main.id   # 자동으로 vpc 먼저 생성
}
```

명시적 (드물게 필요):
```hcl
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy_attachment.s3]
}
```

## 8. Sensitive 데이터

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_url" {
  value     = "postgres://${var.db_password}@..."
  sensitive = true
}
```

`sensitive`는 plan/apply 출력에서 ***로 마스크. 단 **state 파일에는 평문 저장** — 3주차에서 다룸.

## 9. 빅테크 패턴 — module-first

빅테크는 단일 main.tf 운영 거의 없음. 처음부터 모듈 단위로 작성. 4주차 주제.

## 10. 실습

```bash
mkdir tf-demo && cd tf-demo
# 위 main.tf 작성
terraform init
terraform plan -out=tfplan
terraform apply tfplan
# AWS 콘솔에서 bucket 확인
terraform destroy
```

확장:
1. variable 추가 → `terraform plan -var="region=us-east-1"`.
2. data source로 default VPC 조회 → output 출력.
3. `for_each` 로 3 region에 같은 bucket 만들기.

## 11. 자가평가 퀴즈

### Q1. Terraform lifecycle 4단계 순서?
1. **init → plan → apply → destroy**
2. plan → init → apply
3. apply → plan
4. 무관

**정답: 1.**

### Q2. `count`와 `for_each` 중 안전한 추가/삭제?
1. count
2. **for_each (키 기반)**
3. 같음
4. 무관

**정답: 2.**

### Q3. Data source 의 용도?
1. **기존 자원 조회 (생성 X)**
2. 새 자원 생성
3. state 백업
4. 무관

**정답: 1.**

### Q4. `sensitive = true` 효과와 한계?
1. **plan 출력 마스크, 단 state엔 평문**
2. 완전 암호화
3. UI
4. 무관

**정답: 1.**

### Q5. plan 결과를 PR에 첨부하는 이유?
1. **사람이 변경 사항을 사전 검토 → 잘못된 destroy 방지**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 3: State 깊게]에서 backend, locking, workspaces, import를 다룹니다.
