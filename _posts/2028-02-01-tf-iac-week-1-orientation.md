---
layout: post
title: "Infrastructure as Code with Terraform: Week 1 - Orientation & Philosophy"
---

안녕하세요, 클라우드 아키텍트 여러분.

저는 지난 30년간 물리 서버의 랜선을 꽂던 시절부터, 클릭 한 번으로 전 세계에 데이터센터를 띄우는 클라우드 시대까지 인프라의 변천사를 최전선에서 목격했습니다.

오늘날 인프라 운영의 핵심은 **"코드(Code)"**입니다.
서버를 수동으로 만들고, 방화벽을 클릭해서 여는 시대는 끝났습니다.
**IaC (Infrastructure as Code)**는 인프라를 소프트웨어처럼 버전 관리하고, 테스트하고, 배포하는 방법론입니다.

이 과정은 Terraform 문법만 가르치는 것이 아닙니다.
**"Big Tech 기업들은 수만 개의 리소스를 어떻게 코드로 관리하는가?"**에 대한 모범 사례(Best Practice)를 전수하는 과정입니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리는 단순히 EC2 하나를 띄우는 것이 아니라, **IAM, 네트워크, 컨테이너, 보안, 감사가 통합된 프로덕션 레벨의 인프라**를 설계합니다.

### Part 1: Core Fundamentals (핵심 기초)
*   **W1:** 오리엔테이션 & IaC 철학 (Mutable vs Immutable)
*   **W2:** State Management & Backends (협업의 핵심)
*   **W3:** Modules & Reusability (DRY 원칙)

### Part 2: Building the Foundation (기반 구축)
*   **W4:** Identity & Access Management (IAM as Code)
*   **W5:** Networking Infrastructure (VPC, Subnet, Routing)
*   **W6:** Compute & Container Platforms (EC2, EKS/GKE)

### Part 3: Advanced Operations (고급 운영)
*   **W7:** State Manipulation (Import, Refactoring, Workspaces)
*   **W8:** Testing & Validation (Terratest, Checkov)
*   **W9:** CI/CD & Automation (GitOps, Atlantis)

### Part 4: Enterprise Strategy (엔터프라이즈 전략)
*   **W10:** Multi-Cloud & Hybrid Patterns
*   **W11:** Security & Compliance as Code (Sentinel, OPA)
*   **W12:** Capstone - Global Scale Infrastructure Deployment

---

## 🎓 1주차 강의: Why Infrastructure as Code?

### 1. The "ClickOps" Problem
AWS 콘솔에서 마우스로 서버를 만듭니다. 편하죠.
하지만 1년 뒤, **"이 보안 그룹 누가 열었어?"**라고 물으면 아무도 모릅니다.
**"똑같은 환경 하나 더 만들어줘"**라고 하면 며칠이 걸립니다. 그리고 실수로 설정을 빼먹습니다.

### 2. Infrastructure as Code (IaC)
인프라의 상태를 코드로 정의합니다.
*   **Version Control:** 누가 언제 무엇을 바꿨는지 Git에 다 남습니다.
*   **Reproducibility:** 코드만 실행하면 똑같은 환경이 10분 만에 만들어집니다.
*   **Documentation:** 코드 그 자체가 문서입니다.

### 3. Terraform: The Industry Standard
HashiCorp가 만든 오픈소스 도구입니다.
*   **Cloud Agnostic:** AWS, Azure, GCP, 심지어 GitHub, Datadog까지 모든 것을 관리합니다.
*   **Declarative (선언적):** "서버를 만들어라(Imperative)"가 아니라 **"서버가 있어야 한다(Declarative)"**라고 정의합니다. Terraform이 알아서 현재 상태와 비교해 생성/수정/삭제를 결정합니다.

---

## 🛠️ Lab: Setup & First Deploy

### 1. Tools Installation
*   **Terraform:** `brew install terraform` (또는 공식 바이너리)
*   **AWS CLI:** `aws configure` (IAM User 생성 후 Access Key 등록)
*   **IDE:** VS Code + HashiCorp Terraform Extension (필수!)

### 2. Hello World (`main.tf`)
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name-12345"
  
  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```

### 3. The Workflow
1.  `terraform init`: 프로바이더 플러그인 다운로드.
2.  `terraform plan`: **"내가 뭘 할 건지 미리 보여줄게."** (가장 중요!)
3.  `terraform apply`: 실제로 적용.
4.  `terraform destroy`: 싹 다 삭제.

---

## 📝 1주차 과제: Provider Exploration

**목표:** Terraform이 클라우드뿐만 아니라 다양한 SaaS도 관리할 수 있음을 이해하세요.

1.  **Terraform Registry**에 접속하세요.
2.  AWS 외에 관심 있는 프로바이더 3가지를 찾으세요. (예: GitHub, Spotify, Domino's Pizza...)
3.  각 프로바이더로 어떤 리소스(Resource)를 관리할 수 있는지 1줄로 요약하여 제출하세요.
    *   예: **GitHub Provider** - 레포지토리 생성, 팀원 초대, Branch Protection Rule 설정 가능.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Terraform의 언어적 특성 중, "어떻게(How) 만들지"가 아니라 "무엇이(What) 필요한지"를 정의하는 방식을 무엇이라 하나요? (Declarative - 선언적)
2.  **Q2.** Terraform 프로젝트를 처음 시작할 때, 필요한 프로바이더 플러그인을 다운로드하고 초기화하는 명령어는? (`terraform init`)
3.  **Q3.** 실제 인프라를 변경하기 전에, 변경 사항을 미리 시뮬레이션하여 보여주는 명령어는? (`terraform plan`)

---

다음 주, Terraform의 심장인 **State(상태 파일)**에 대해 배웁니다.
이 `tfstate` 파일을 잃어버리면 여러분의 인프라는 미아가 됩니다. 협업을 위한 **Remote Backend** 설정법을 익혀봅시다.

**Code your world.**
