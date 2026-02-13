---
layout: post
title: "Infrastructure as Code with Terraform: Week 8 - Testing & Validation (Terratest, OPA/Sentinel)"
---

소프트웨어 개발에는 유닛 테스트가 있는데, 인프라에는 왜 없나요?
있습니다. 인프라도 코드니까요.

"S3 버킷이 Public으로 열려있으면 배포 막아!"
"EC2 인스턴스 타입이 `t3.large`보다 크면 에러 내!"
이런 정책 검사를 자동화합니다.

---

## 1. Unit Testing vs Integration Testing

*   **Unit Test (Static Analysis):** `terraform plan` 결과를 분석. 리소스를 실제로 안 만듦. 빠름. (OPA, Sentinel, Checkov)
*   **Integration Test:** 실제로 `terraform apply`를 해서 리소스를 만들고, `curl`을 날려서 확인하고, `terraform destroy` 함. 느리고 돈 듦. (Terratest)

---

## 2. Policy as Code (OPA / Sentinel)

**OPA (Open Policy Agent)**는 JSON/YAML을 검사하는 정책 엔진입니다. `Rego`라는 언어를 씁니다.

**예제 (Rego):**
```rego
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.after.acl == "public-read"
  msg := "Public S3 bucket is forbidden!"
}
```
`terraform plan -out tfplan.binary` -> `terraform show -json tfplan.binary` -> OPA 검사.

---

## 3. Terratest (Go)

Go 언어로 작성하는 통합 테스트 프레임워크입니다.

```go
func TestTerraformAwsS3(t *testing.T) {
	terraformOptions := &terraform.Options{
		TerraformDir: "../examples/s3",
	}

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	bucketName := terraform.Output(t, terraformOptions, "bucket_name")
	aws.AssertS3BucketExists(t, "us-east-1", bucketName)
}
```

---

## 🛠️ Lab: Static Analysis with Checkov

OPA는 배우기 어렵습니다. 대신 사용하기 쉬운 **Checkov**를 써봅니다.

1.  `pip install checkov`
2.  의도적으로 취약한 코드(`aws_security_group`에 `0.0.0.0/0` 오픈) 작성.
3.  `checkov -d .` 실행.
4.  빨간색으로 실패 메시지가 뜨는 것을 확인.
5.  `#checkov:skip=CKV_AWS_1:Justification` 주석으로 예외 처리 해보기.

---

## 📝 8주차 과제: Validation Rules

**목표:** Terraform 내장 기능인 `variable validation`과 `precondition`을 사용하여 방어 코드를 작성하세요.

1.  `instance_type` 변수에 `t2.micro` 또는 `t3.micro`만 입력받도록 정규식(`regex`) 검증을 넣으세요.
2.  `lifecycle { precondition { ... } }` 블록을 사용하여, AMI ID가 `ami-`로 시작하지 않으면 에러를 뱉게 만드세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Terraform Plan 결과를 JSON으로 변환하여, 특정 정책(예: 모든 리소스에 태그 필수)을 위반했는지 검사하는 도구의 이름은? (OPA, Sentinel, Checkov 등)
2.  **Q2.** 실제로 인프라를 배포하고(HTTP 요청 등) 테스트한 뒤 삭제하는 방식의 테스트 프레임워크는? (Terratest)
3.  **Q3.** Terraform 1.2부터 도입된 기능으로, 리소스 생성 전 특정 조건을 만족하는지 검사하고 실패 시 커스텀 에러 메시지를 띄우는 블록은? (`precondition` / `postcondition`)

---

다음 주, 테스트까지 마친 코드를 자동으로 배포하는 **CI/CD 파이프라인**을 구축합니다.
PR을 올리면 봇이 `terraform plan` 결과를 댓글로 달아줍니다.

**Trust, but Verify.**
