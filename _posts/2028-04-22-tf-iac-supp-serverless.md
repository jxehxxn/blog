---
layout: post
title: "Infrastructure as Code with Terraform: Supplement - Serverless Infrastructure with Terraform"
---

서버리스(Serverless)는 "서버가 없는 게" 아니라, "서버 관리를 안 하는 것"입니다.
Terraform으로 EC2를 띄우는 것과 Lambda를 띄우는 것은 접근 방식이 완전히 다릅니다.

---

## 1. The Serverless Stack

AWS 기준, 서버리스 3대장은 다음과 같습니다.
1.  **Lambda:** 컴퓨팅 (코드 실행).
2.  **API Gateway:** 문지기 (HTTP 요청 처리).
3.  **DynamoDB:** 저장소 (NoSQL).

이 셋을 Terraform으로 엮으려면 수많은 리소스가 필요합니다.
(Lambda Function, IAM Role, Policy, Log Group, API Gateway REST API, Resource, Method, Integration, Deployment...)

---

## 2. Managing Lambda Code

Terraform은 인프라 도구지, 배포 도구는 아닙니다.
하지만 Lambda 코드가 바뀌면 배포도 해야 하죠.

### `archive_file` Data Source
Terraform이 소스 코드를 Zip으로 압축해서 업로드합니다.

```hcl
data "archive_file" "lambda" {
  type        = "zip"
  source_file = "index.js"
  output_path = "lambda.zip"
}

resource "aws_lambda_function" "test" {
  filename         = "lambda.zip"
  source_code_hash = data.archive_file.lambda.output_base64sha256
  # ...
}
```
**팁:** `source_code_hash`가 바뀌어야만 Terraform이 "아, 코드가 바꼈네?" 하고 다시 배포합니다.

---

## 3. Serverless Framework vs Terraform

"서버리스 프레임워크(Serverless.com)가 더 편하지 않나요?"
맞습니다. 개발자는 그게 편합니다.
하지만 인프라 팀 입장에서는 **VPC, IAM, DB 등 전체 인프라와 통합 관리**하기 위해 Terraform을 선호합니다.

**하이브리드 전략:**
*   **Terraform:** DynamoDB, IAM Role, VPC 등 "변하지 않는 인프라" 관리.
*   **Serverless Framework:** Lambda 함수 코드와 API Gateway 등 "자주 변하는 앱" 배포.

---

## 4. Terraform Module for Serverless

`terraform-aws-modules/lambda/aws` 모듈을 쓰면 Serverless Framework만큼 편해집니다.

```hcl
module "lambda_function" {
  source = "terraform-aws-modules/lambda/aws"

  function_name = "my-lambda"
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  source_path   = "./src"

  attach_policy_json = true
  policy_json        = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = ["s3:GetObject"]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}
```
이 모듈이 압축, 업로드, IAM 생성, 로그 그룹 생성을 다 해줍니다.

---

## 🛠️ Lab: A Serverless API

Terraform 만으로 "Hello World" API를 만들어봅니다.

1.  **Lambda:** Node.js 코드를 작성하고 `archive_file`로 압축.
2.  **IAM:** Lambda가 로그를 남길 수 있게(`logs:*`) Role 생성.
3.  **Function URL:** API Gateway 대신 간단하게 `aws_lambda_function_url` 리소스를 사용하여 HTTP 엔드포인트를 엽니다. (Terraform 1.2+ / AWS Provider 4.0+ 기능)
4.  **테스트:** 생성된 URL로 `curl`을 날려 응답 확인.

---

## 5. Summary

서버리스는 인프라 관리가 없어서 편하지만, **코드와 인프라의 경계가 모호**합니다.
Terraform을 사용하면 이 경계를 명확히 하고, 기존의 거대한 클라우드 인프라 속에 서버리스를 매끄럽게 통합할 수 있습니다.
