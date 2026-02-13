---
layout: post
title: "Infrastructure as Code with Terraform: Week 4 - Managing IAM with Terraform (AWS/GCP/Azure)"
---

"개발자 김철수에게 S3 읽기 권한을 주세요."
콘솔에서 클릭, 클릭, 저장.
이렇게 하면 나중에 **"김철수가 무슨 권한 가지고 있어?"**라고 물었을 때 대답 못 합니다.

**IAM as Code**는 권한 관리를 투명하고 감사 가능(Auditable)하게 만듭니다.
특히 IAM은 보안 사고와 직결되므로, 가장 먼저 코드로 관리해야 할 대상입니다.

---

## 1. IAM Principles (Least Privilege)

**최소 권한의 원칙:** 딱 필요한 만큼만 권한을 줘라.
*   **Bad:** `AdministratorAccess` (귀찮으니까 다 줌)
*   **Good:** `s3:GetObject` on `arn:aws:s3:::my-bucket/*`

---

## 2. Defining IAM Policy in Terraform

JSON 파일을 문자열로 넣는 건 구립니다. `aws_iam_policy_document` 데이터 소스를 쓰세요.

```hcl
data "aws_iam_policy_document" "s3_read" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["arn:aws:s3:::my-bucket/*"]
    effect    = "Allow"
  }
}

resource "aws_iam_policy" "policy" {
  name   = "s3-read-policy"
  policy = data.aws_iam_policy_document.s3_read.json
}
```
이렇게 하면 구문 오류도 잡아주고 가독성도 좋습니다.

---

## 3. Users, Groups, and Roles

*   **User:** 사람. (AWS Console 로그인) -> **가능하면 만들지 마세요.** (SSO 권장)
*   **Group:** 유저들의 집합. (Developers, Admins)
*   **Role:** 모자(Hat). EC2가 쓰거나, 다른 계정에서 넘어올 때 씀.

**Best Practice:**
사람에게 직접 Policy를 붙이지 마세요. **Group에 Policy를 붙이고, 사람을 Group에 넣으세요.**

```hcl
resource "aws_iam_group_policy_attachment" "test-attach" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.policy.arn
}
```

---

## 4. Assume Role (Cross-Account Access)

운영(Prod) 계정과 개발(Dev) 계정이 분리되어 있다면?
개발 계정의 유저가 운영 계정의 Role을 **Assume(가면 쓰기)**해서 작업합니다.
이 복잡한 신뢰 관계(Trust Relationship)도 Terraform 코드로 관리하면 한눈에 보입니다.

---

## 🛠️ Lab: IAM Strategy Implementation

1.  **Group 생성:** `Developers`, `Ops`
2.  **Policy 생성:**
    *   `Developers`: EC2 ReadOnly, S3 FullAccess (특정 버킷만)
    *   `Ops`: AdministratorAccess
3.  **User 생성:** `alice`, `bob`
4.  **할당:** Alice -> Developers, Bob -> Ops
5.  **검증:** 콘솔에서 Policy Simulator를 돌리거나, 실제 로그인해서 권한을 확인합니다.

---

## 📝 4주차 과제: Service Account Management

**목표:** 애플리케이션(EC2, Lambda)을 위한 IAM Role을 생성하세요.

1.  EC2가 S3에 사진을 업로드할 수 있는 Role을 만듭니다.
2.  Trust Policy(신뢰 관계)에 `ec2.amazonaws.com`이 이 Role을 쓸 수 있게 허용합니다.
3.  Instance Profile을 생성하여 EC2와 연결합니다.
4.  이 모든 과정을 Terraform 코드로 작성하여 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** JSON 형태의 IAM 정책을 HCL 문법으로 더 안전하고 가독성 있게 작성하게 도와주는 Terraform Data Source는? (aws_iam_policy_document)
2.  **Q2.** IAM User에게 직접 권한을 부여하는 것보다 IAM Group을 사용하는 것이 권장되는 이유는? (관리 효율성 - 입/퇴사 시 그룹 멤버십만 변경하면 됨)
3.  **Q3.** EC2 인스턴스에 IAM Role을 연결하기 위해 필요한 중간 매개체 리소스는? (IAM Instance Profile)

---

다음 주, 클라우드의 척추인 **네트워크(VPC)**를 구축합니다.
IP 계산기(CIDR) 준비하세요.

**Identity is the new perimeter.**
