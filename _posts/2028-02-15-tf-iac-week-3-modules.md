---
layout: post
title: "Infrastructure as Code with Terraform: Week 3 - Modules & Reusability"
---

프로젝트가 커지면 `main.tf`가 5000줄이 넘어갑니다. 관리 불가능이죠.
프로그래밍에서 함수(Function)를 만들어 코드를 재사용하듯이, Terraform에서는 **모듈(Module)**을 사용합니다.

"웹 서버"라는 모듈을 만들어두면,
`module "web_server_a" {...}`
`module "web_server_b" {...}`
이렇게 쉽게 찍어낼 수 있습니다.

---

## 1. What is a Module?

사실 모든 Terraform 폴더는 그 자체로 모듈입니다(Root Module).
하지만 보통 **"다른 폴더에 있는 코드를 불러다 쓰는 것"**을 모듈이라고 합니다.

### Structure
```
modules/
  aws-vpc/          # Child Module
    main.tf
    variables.tf    # 입력 (함수의 파라미터)
    outputs.tf      # 출력 (함수의 리턴값)
main.tf             # Root Module (여기서 호출)
```

---

## 2. Using Modules

### Local Module
내 컴퓨터에 있는 폴더를 참조합니다.
```hcl
module "my_vpc" {
  source = "./modules/aws-vpc"
  
  cidr_block = "10.0.0.0/16" # 변수 전달
}
```

### Remote Module (Terraform Registry)
남이 잘 만들어둔 공인 모듈을 씁니다. (AWS 공식 모듈 등)
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  ...
}
```
**Tip:** 처음엔 직접 짜보다가, 나중엔 검증된 커뮤니티 모듈을 쓰는 게 정신건강에 좋습니다.

---

## 3. Module Design Patterns

좋은 모듈을 만드는 원칙:
1.  **Encapsulation:** 복잡한 내부 구현(보안 그룹 규칙, 라우팅 테이블 등)은 숨기고, 사용자는 `cidr` 같은 간단한 변수만 넣게 합니다.
2.  **Opinionated:** 우리 회사의 보안 정책(예: 모든 버킷은 암호화 필수)을 모듈 안에 강제해버립니다. 개발자들은 이 모듈만 쓰면 자동으로 보안 규정을 지키게 됩니다.

---

## 🛠️ Lab: Create a Web Server Module

EC2 인스턴스와 보안 그룹을 묶어서 `web-server` 모듈을 만들어봅시다.

1.  `modules/web-server/main.tf`: `aws_instance`와 `aws_security_group` 생성.
2.  `modules/web-server/variables.tf`: `instance_type`, `ami_id` 등을 변수로 선언.
3.  `modules/web-server/outputs.tf`: 생성된 인스턴스의 `public_ip` 출력.
4.  `main.tf` (Root): 위 모듈을 호출하여 `dev-web`, `prod-web` 두 개 생성.

---

## 📝 3주차 과제: Refactoring to Modules

**목표:** Week 1~2에서 짠 단일 파일 코드를 모듈 구조로 리팩토링하세요.

1.  `s3-bucket` 모듈 생성. (버킷 생성 + 태깅 + 암호화 설정 포함)
2.  `dynamodb-table` 모듈 생성.
3.  루트 `main.tf`는 이 두 모듈을 호출하는 코드로만 구성되도록 정리하세요.
4.  `terraform plan` 실행 시 리소스가 변경/삭제되지 않고 그대로 유지되어야 합니다(State가 같다면).

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Terraform Module의 3가지 구성 요소 파일은? (main.tf, variables.tf, outputs.tf)
2.  **Q2.** 로컬 경로가 아닌 GitHub 리포지토리에 있는 모듈을 소스로 사용할 수 있나요? (네, `source = "github.com/..."` 형식으로 가능)
3.  **Q3.** 모듈 내부의 리소스 속성(예: EC2 ID)을 모듈 외부(Root)에서 참조하려면 어떻게 해야 하나요? (모듈 내 `outputs.tf`에 정의해야 함)

---

이것으로 기초(Part 1)가 끝났습니다.
다음 주, Part 2의 시작은 클라우드의 문지기, **IAM (Identity and Access Management)**입니다. 권한 관리도 코드로 합니다.

**DRY (Don't Repeat Yourself).**
