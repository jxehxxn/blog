---
layout: post
title: "Infrastructure as Code with Terraform: Week 10 - Multi-Cloud & Hybrid Strategies"
---

"우리는 AWS만 써요."
하지만 언젠가 GCP의 빅데이터를 써야 할 수도 있고, Azure의 AD를 써야 할 수도 있습니다.
Terraform의 가장 큰 장점은 **Provider만 갈아끼우면 어디든 간다**는 것입니다.

---

## 1. Provider Aliases

하나의 `main.tf`에서 서울 리전(ap-northeast-2)과 버지니아 리전(us-east-1)에 동시에 리소스를 만들려면?
`alias`를 씁니다.

```hcl
provider "aws" {
  region = "ap-northeast-2"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

resource "aws_s3_bucket" "seoul" { ... }

resource "aws_s3_bucket" "virginia" {
  provider = aws.us # 여기!
  ...
}
```

---

## 2. Multi-Cloud Pattern

AWS의 데이터를 GCP BigQuery로 분석하고 싶다면?

1.  **AWS S3:** 데이터 저장.
2.  **GCP Storage Transfer Service:** S3에서 데이터를 가져옴.
3.  **Terraform:** 이 둘을 한 코드에서 정의.

```hcl
resource "aws_s3_bucket" "data" { ... }

resource "google_storage_transfer_job" "sync" {
  transfer_spec {
    aws_s3_data_source {
      bucket_name = aws_s3_bucket.data.bucket
    }
  }
}
```
**팁:** 서로 다른 클라우드 간의 인증(Authentication) 처리가 가장 까다롭습니다. (Workload Identity Federation 권장)

---

## 3. Terraform Cloud / Enterprise

상태 관리, CI/CD, 정책 검사(Sentinel)를 통합 제공하는 SaaS입니다.
기업 규모가 커지면 자체 구축(Atlantis)보다 이걸 사는 게 쌀 수도 있습니다.

---

## 🛠️ Lab: Multi-Region Deployment

CloudFront(CDN)는 전 세계에 배포되지만, 인증서(ACM)는 반드시 `us-east-1`에 있어야 합니다.
이 제약사항을 `alias` 프로바이더로 해결해봅니다.

1.  기본 프로바이더: `ap-northeast-2`
2.  Alias 프로바이더: `us-east-1`
3.  `aws_acm_certificate` 리소스를 만들 때 `provider = aws.us`를 지정.
4.  `terraform apply`로 서울 리전 리소스와 버지니아 리전 리소스가 동시에 생성되는지 확인.

---

## 📝 10주차 과제: Hybrid Connectivity

**목표:** 온프레미스 데이터센터와 AWS VPC를 잇는 **Site-to-Site VPN** 구성을 Terraform으로 설계하세요. (실제 장비가 없으니 코드만 작성)

1.  `aws_vpn_gateway` (AWS 측)
2.  `aws_customer_gateway` (온프레미스 측 IP)
3.  `aws_vpn_connection` (터널링)
4.  온프레미스 장비(Cisco/Juniper) 설정 파일은 Terraform이 못 만듭니다. 하지만 설정값을 `output`으로 출력해줄 수는 있습니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 하나의 Terraform 설정 파일 내에서 서로 다른 두 개의 AWS 리전이나 계정을 다루기 위해 Provider 블록에 추가해야 하는 설정은? (`alias`)
2.  **Q2.** 서로 다른 클라우드 벤더(AWS-GCP) 간에 데이터를 전송하거나 연동할 때, 장기 Access Key를 교환하지 않고 안전하게 인증하는 방식은? (Workload Identity Federation / OIDC)
3.  **Q3.** Terraform Cloud(TFC)의 기능 중, 로컬 컴퓨터가 아닌 TFC 서버에서 `terraform apply`를 실행하게 해주는 기능은? (Remote Execution)

---

다음 주, 보안 팀이 제일 좋아하는 주제입니다.
**"보안 규정을 코드로 강제하는 법"**을 배웁니다.

**One tool to rule them all.**
