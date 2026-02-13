---
layout: post
title: "Infrastructure as Code with Terraform: Week 9 - CI/CD for Infrastructure (GitHub Actions, Atlantis)"
---

로컬에서 `terraform apply`를 하는 건 혼자 할 때나 가능한 일입니다.
팀 프로젝트에서는 **"누가, 언제, 무엇을 배포했는지"** 기록이 남아야 하고, **승인 절차(Approval)**가 있어야 합니다.

이것이 **GitOps**입니다.

---

## 1. The GitOps Flow

1.  개발자가 코드 수정 후 PR 생성.
2.  **CI 봇:** `terraform plan` 실행 -> 결과를 PR 댓글로 남김.
3.  **Reviewer:** 댓글을 보고 변경 사항 확인(Review) -> 승인(Approve).
4.  **Merge:** 코드가 머지되면 **CD 봇**이 `terraform apply` 실행.

---

## 2. GitHub Actions Workflow

가장 대중적인 방법입니다.

```yaml
# .github/workflows/terraform.yml
on: [pull_request]
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: hashicorp/setup-terraform@v2
      - run: terraform init
      - run: terraform plan -no-color > plan.txt
      - uses: actions/github-script@v6
        with:
          script: |
            // plan.txt 내용을 읽어서 PR 코멘트로 달기
```

---

## 3. Atlantis: Pull Request Automation

Terraform 전용 CI/CD 도구입니다.
PR 댓글에 `atlantis plan`이라고 치면 봇이 Plan을 떠오고, `atlantis apply`라고 치면 배포합니다.

*   **Locking:** PR이 열려있는 동안 해당 폴더에 Lock을 걸어 다른 사람의 충돌을 방지합니다.
*   **Approval:** GitHub의 "Required Reviews" 설정과 연동됩니다.

---

## 🛠️ Lab: GitHub Actions Pipeline

1.  GitHub Repo 생성.
2.  AWS 자격 증명(`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)을 GitHub Secrets에 등록.
3.  `terraform fmt -check` (포맷 검사)와 `terraform plan`을 수행하는 워크플로우 작성.
4.  일부러 틀린 문법으로 PR을 올려서 CI가 실패하는지 확인.

---

## 📝 9주차 과제: OIDC Authentication

**목표:** GitHub Actions에 AWS Access Key(장기 자격 증명)를 저장하지 않고, **OIDC(OpenID Connect)**를 통해 임시 자격 증명을 받아오게 구성하세요.

1.  AWS IAM에 OIDC Identity Provider를 생성합니다 (token.actions.githubusercontent.com).
2.  GitHub Repo의 특정 Branch만 Assume 할 수 있는 IAM Role을 만듭니다.
3.  GitHub Actions에서 `aws-actions/configure-aws-credentials` 액션을 사용하여 Role을 Assume 하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** CI/CD 파이프라인에서 `terraform apply` 시 사용자 입력을 기다리지 않고 자동으로 'Yes' 처리하기 위해 사용하는 플래그는? (`-auto-approve`)
2.  **Q2.** GitHub Actions에 AWS 장기 자격 증명(Access Key)을 저장하는 것보다 OIDC를 사용하는 것이 권장되는 이유는? (키 유출 위험이 없고, 임시 자격 증명을 사용하여 보안성이 높음)
3.  **Q3.** Atlantis와 같은 도구가 제공하는 기능으로, 한 번에 하나의 PR만 특정 상태를 변경할 수 있도록 막는 기능은? (State Locking / Project Locking)

---

자동화까지 끝났습니다.
이제 시야를 넓혀 **Multi-Cloud**와 **Enterprise Strategy**를 고민해봅시다.

**Automate the boring stuff.**
