---
layout: post
title: "Infrastructure as Code with Terraform: Week 2 - Terraform State Management & Backends"
---

"Terraform은 내가 만든 서버를 어떻게 기억할까요?"
정답은 **State File (`terraform.tfstate`)**입니다.

이 파일은 Terraform의 **생명**입니다.
이걸 로컬 노트북에만 저장하면? 노트북 잃어버리면 인프라 관리 못 합니다.
팀원 A가 수정하고 팀원 B가 수정하면? 충돌 나서 파일이 깨집니다.

그래서 실무에서는 반드시 **Remote Backend**를 씁니다.

---

## 1. What is State?

Terraform이 관리하는 리소스들의 현재 상태를 기록한 JSON 파일입니다.
*   **Mapping:** 코드의 `resource "aws_instance" "web"`이 실제 AWS의 `i-12345abcde` 인스턴스라는 것을 매핑합니다.
*   **Metadata:** 의존성 정보, 리소스 속성 등을 저장합니다.

---

## 2. Remote Backend (S3 + DynamoDB)

혼자 할 땐 로컬 파일(`local`)도 괜찮지만, 팀 프로젝트는 무조건 원격 저장소(`remote`)를 써야 합니다.
AWS에서는 **S3 (저장소)**와 **DynamoDB (잠금 장치)** 조합이 표준입니다.

*   **S3 Bucket:** `.tfstate` 파일을 안전하게 저장. (Versioning 필수!)
*   **DynamoDB Table:** **Locking**을 담당. 누군가 `terraform apply` 중이면 다른 사람은 건드리지 못하게 막음.

### Configuration (`backend.tf`)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock-table"
    encrypt        = true
  }
}
```

---

## 3. State Locking

**시나리오:**
1.  철수가 `terraform apply`를 실행 중입니다. (인프라 변경 중)
2.  영희가 동시에 `terraform apply`를 실행합니다.
3.  Lock이 없다면? **State 파일이 꼬여서 인프라가 망가집니다.**

DynamoDB를 설정하면, 철수가 실행할 때 테이블에 "나 작업 중임"이라고 기록(Lock)합니다. 영희는 "Lock 걸려있네?" 하고 대기하거나 에러를 뱉습니다.

---

## 🛠️ Lab: Configuring Remote Backend

**닭이 먼저냐 알이 먼저냐 문제(Chicken and Egg Problem):**
S3 버킷과 DynamoDB 테이블은 누가 만드나요?
처음 한 번은 로컬 상태로 S3/DynamoDB를 만들고, 그 다음에 Backend 설정을 추가해서 마이그레이션해야 합니다.

1.  `main.tf`에 `aws_s3_bucket`과 `aws_dynamodb_table` 리소스 작성.
2.  `terraform apply` (로컬 상태로 생성됨).
3.  `backend.tf` 작성 (방금 만든 버킷/테이블 정보 입력).
4.  `terraform init` -> **"Do you want to copy existing state to the new backend?"** -> **Yes**.
5.  로컬의 `terraform.tfstate`를 지워도 `terraform plan`이 잘 되는지 확인.

---

## 📝 2주차 과제: State Isolation Strategy

**목표:** 개발(Dev) 환경과 운영(Prod) 환경의 State를 어떻게 격리할지 전략을 세우세요.

1.  **전략 A:** 폴더 격리 (`dev/`, `prod/` 폴더를 따로 만들고 각각 `terraform init`).
2.  **전략 B:** Workspace 사용 (`terraform workspace new dev`, `new prod`).
3.  두 전략의 장단점을 조사하여 비교 리포트를 작성하세요. (대부분의 기업은 전략 A를 선호합니다. 왜일까요?)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Terraform State 파일이 손상되거나 삭제되었을 때 발생할 수 있는 가장 큰 문제는? (Terraform이 실제 리소스를 추적하지 못해, 리소스를 중복 생성하거나 관리 불능 상태가 됨)
2.  **Q2.** S3를 백엔드로 사용할 때, 동시 실행을 방지하기 위해 State Locking 기능을 제공하는 AWS 서비스는? (DynamoDB)
3.  **Q3.** 팀원들과 협업 시 State 파일에 민감한 정보(비밀번호 등)가 평문으로 저장되는 것을 방지하기 위해 S3 버킷에 적용해야 하는 설정은? (Server-Side Encryption)

---

다음 주, 코드를 복사-붙여넣기 하지 않고 재사용하는 **Module**에 대해 배웁니다.
"VPC 만드는 코드"를 한 번만 짜놓고, 100번 돌려막는 마법입니다.

**State is Truth.**
