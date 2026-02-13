---
layout: post
title: "Infrastructure as Code with Terraform: Supplement - HCL Fundamentals"
---

Week 2로 넘어가기 전에, Terraform의 언어인 **HCL (HashiCorp Configuration Language)** 문법을 확실히 잡고 가야 합니다.
JSON과 비슷하지만 다르고, YAML보다 강력합니다.

---

## 1. Blocks & Arguments

HCL은 블록 구조로 되어 있습니다.

```hcl
# <BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
#   <IDENTIFIER> = <EXPRESSION>
# }

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```
*   `resource`: 블록 타입 (data, variable, output 등).
*   `aws_instance`: 리소스 종류 (Provider가 정의).
*   `web`: 내가 지은 이름 (Local Name). 다른 곳에서 참조할 때 씀.

---

## 2. Variables & Outputs

입력(Input)과 출력(Output)입니다.

### Input Variables (`variables.tf`)
```hcl
variable "region" {
  type        = string
  default     = "us-east-1"
  description = "AWS Region to deploy"
}
```

### Outputs (`outputs.tf`)
```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

---

## 3. Interpolation & References

다른 리소스의 값을 가져다 쓸 때가 핵심입니다.

```hcl
resource "aws_eip" "lb" {
  instance = aws_instance.web.id # 위에서 만든 web 인스턴스의 ID를 참조
  vpc      = true
}
```
**의존성 자동 관리:** Terraform은 이 참조를 보고 "아, `aws_instance.web`을 먼저 만들고 나서 `aws_eip`를 만들어야겠구나"라고 순서를 결정합니다. (Dependency Graph)

---

## 4. Functions

Terraform은 강력한 내장 함수를 제공합니다.

*   `cidrsubnet("10.0.0.0/16", 8, 1)` -> `"10.0.1.0/24"` (네트워크 계산)
*   `file("user_data.sh")` -> 파일 내용 읽기
*   `jsonencode({...})` -> 객체를 JSON 문자열로 변환 (IAM 정책 만들 때 필수)

---

## 5. Meta-Arguments

모든 리소스에 공통으로 쓸 수 있는 특수 설정입니다.

*   `count`: 리소스를 N개 만들 때. (`count = 3`)
*   `for_each`: 리스트나 맵을 돌면서 만들 때. (더 권장됨)
*   `depends_on`: 명시적으로 의존성을 걸 때. (Terraform이 순서를 못 찾을 때만 사용)
*   `lifecycle`: 리소스 수명 주기 관리. (`create_before_destroy`, `prevent_destroy`)

이 문법들이 손에 익어야 "인프라 프로그래밍"이 가능해집니다.
