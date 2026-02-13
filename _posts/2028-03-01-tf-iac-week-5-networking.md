---
layout: post
title: "Infrastructure as Code with Terraform: Week 5 - Networking Infrastructure (VPC, Subnets, Firewalls)"
---

네트워크는 한 번 만들면 바꾸기 정말 어렵습니다.
IP 대역을 잘못 잡으면? VPC를 다 뜯어고쳐야 합니다.
그래서 네트워크야말로 **IaC의 꽃**입니다. 철저한 계획과 모듈화가 필수입니다.

---

## 1. VPC Architecture Design

표준적인 3-Tier 아키텍처를 그립니다.

*   **VPC:** `10.0.0.0/16` (65,536개 IP)
*   **Public Subnet:** `10.0.1.0/24`, `10.0.2.0/24` (ALB, Bastion)
*   **Private Subnet:** `10.0.10.0/24`, `10.0.11.0/24` (App Server)
*   **Database Subnet:** `10.0.20.0/24`, `10.0.21.0/24` (RDS)

**Multi-AZ:** 가용 영역(AZ) 2개 이상에 분산 배치해야 합니다.

---

## 2. Terraform Network Patterns

### `cidrsubnet` Function
IP를 손으로 계산하지 마세요. 함수를 쓰세요.

```hcl
cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index)
# 10.0.0.0/16 -> 10.0.0.0/24, 10.0.1.0/24 ...
```

### Route Tables
*   **Public:** Internet Gateway(IGW)로 가는 라우팅 (`0.0.0.0/0 -> igw`).
*   **Private:** NAT Gateway로 가는 라우팅 (`0.0.0.0/0 -> nat`).

---

## 3. Security Groups (Virtual Firewall)

보안 그룹은 **Stateful**합니다. 들어오는 거(Ingress) 허용하면 나가는 거(Egress)는 자동 허용입니다.

**Module Tip:**
"Web SG는 App SG에게 80 포트를 열어준다" 같은 의존성이 있습니다.
Terraform에서 `aws_security_group_rule` 리소스를 따로 분리하면 순환 참조(Circular Dependency)를 피할 수 있습니다.

---

## 🛠️ Lab: Building a VPC from Scratch

`terraform-aws-modules/vpc/aws` 모듈을 쓰지 않고(학습용), 직접 리소스를 하나하나 정의해서 VPC를 만들어봅니다.

1.  `aws_vpc`
2.  `aws_internet_gateway`
3.  `aws_subnet` (Public x 2, Private x 2)
4.  `aws_nat_gateway` (EIP 필요)
5.  `aws_route_table` & `aws_route_table_association`

---

## 📝 5주차 과제: Network Module

**목표:** Lab에서 만든 VPC 코드를 모듈화하고, `terraform-docs`를 이용해 문서를 자동 생성하세요.

1.  `modules/vpc` 폴더로 이동.
2.  입력 변수: `vpc_cidr`, `public_subnets` (List), `private_subnets` (List), `azs` (List).
3.  출력 변수: `vpc_id`, `private_subnet_ids`.
4.  루트에서 이 모듈을 호출하여 `dev-vpc` (10.0.0.0/16)와 `stage-vpc` (10.1.0.0/16)를 피어링(Peering) 없이 생성하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Private Subnet에 있는 EC2가 인터넷에 접속하여 패키지를 다운로드하려면 필요한 게이트웨이는? (NAT Gateway)
2.  **Q2.** Terraform에서 IP 대역(CIDR)을 자동으로 계산해주는 내장 함수 이름은? (`cidrsubnet`)
3.  **Q3.** Security Group과 NACL(Network ACL)의 가장 큰 차이점은? (Stateful vs Stateless)

---

다음 주, 닦아놓은 도로(Network) 위에 건물(Compute)을 올립니다.
**EC2**와 **Kubernetes (EKS)** 클러스터를 코드로 배포합니다.

**Ping success.**
