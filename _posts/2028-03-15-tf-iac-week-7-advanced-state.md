---
layout: post
title: "Infrastructure as Code with Terraform: Week 7 - Advanced State Management (Workspaces, Import, Refactoring)"
---

"이미 손으로 만든 AWS 리소스가 100개 있는데, 이걸 Terraform으로 가져오고 싶어요."
"모듈 구조를 바꾸고 싶은데, `destroy` 없이 옮길 수 있나요?"

Terraform 초보자는 여기서 막힙니다. **State 조작(Manipulation)** 기술은 중급자로 가는 관문입니다.

---

## 1. Importing Existing Infrastructure

`terraform import` 명령어를 씁니다.
하지만 `import`는 **State 파일에만** 기록해줍니다. 코드는 안 만들어줍니다. (Terraform 1.5부터 `import` 블록으로 코드 생성 일부 지원)

### Process
1.  빈 리소스 작성: `resource "aws_s3_bucket" "b" {}`
2.  Import 실행: `terraform import aws_s3_bucket.b my-existing-bucket`
3.  `terraform plan` 실행 -> 차이점(Diff) 확인.
4.  코드 수정 -> `terraform plan` -> "No changes" 뜰 때까지 반복.

---

## 2. Refactoring with `moved` block

리소스 이름을 바꾸거나 모듈로 옮기면, Terraform은 **"삭제 후 재생성"**하려고 합니다. DB라면 큰일 나죠.
Terraform 1.1부터 도입된 `moved` 블록을 쓰면 됩니다.

```hcl
moved {
  from = aws_instance.web
  to   = module.web_server.aws_instance.this
}
```
이러면 Terraform이 "아, 이름만 바뀐 거구나" 하고 State만 수정합니다.

---

## 3. Workspaces

하나의 코드로 여러 환경(Dev, Stage, Prod)을 관리하는 방법입니다.
`terraform workspace new dev`

*   **장점:** 코드가 1개라 관리가 쉽다.
*   **단점:** 실수로 Prod 워크스페이스에서 `destroy` 칠 위험이 크다. State 파일이 하나로 뭉쳐있어 관리가 어렵다.

**결론:** 워크스페이스보다는 **폴더 분리 (`env/dev`, `env/prod`)** 방식을 더 권장합니다.

---

## 🛠️ Lab: The Great Migration

콘솔에서 손으로 만든 리소스를 코드로 가져와 봅니다.

1.  AWS 콘솔에서 S3 버킷을 하나 만듭니다. (태그도 몇 개 붙이세요)
2.  `import.tf` 파일을 만들고 `import` 블록을 작성합니다.
3.  `terraform plan -generate-config-out=generated.tf` (1.5+ 기능) 명령어로 코드를 자동 생성해 봅니다.
4.  생성된 코드를 다듬고 `terraform apply`를 해서 State를 동기화합니다.

---

## 📝 7주차 과제: Zero-Downtime Refactoring

**목표:** `aws_security_group.web` 리소스를 `module.security_group` 내부로 옮기는 리팩토링을 수행하세요.

1.  기존 상태에서 `terraform plan` (No changes).
2.  모듈을 만들고 리소스 정의를 옮깁니다.
3.  `moved` 블록을 작성합니다.
4.  다시 `terraform plan`을 했을 때 **"0 to add, 0 to change, 0 to destroy"**가 나와야 성공입니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 이미 존재하는 인프라 리소스를 Terraform State로 가져오는 명령어는? (`terraform import`)
2.  **Q2.** 리소스의 이름을 변경하거나 모듈로 이동시킬 때, 리소스가 삭제되고 다시 생성되는 것을 방지하기 위해 사용하는 블록은? (`moved`)
3.  **Q3.** `terraform state list`, `terraform state mv`, `terraform state rm` 등의 하위 명령어를 사용할 때 가장 주의해야 할 점은? (State 파일이 손상될 수 있으므로 백업 필수, 협업 중인 경우 Lock 주의)

---

다음 주, "내 코드가 안전한가?"
테스트 코드(Test Code)를 작성하여 인프라를 검증하는 **Terratest**와 **OPA**를 배웁니다.

**State Surgeon.**
