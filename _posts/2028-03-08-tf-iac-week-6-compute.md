---
layout: post
title: "Infrastructure as Code with Terraform: Week 6 - Compute & Containers (EC2, EKS/GKE)"
---

인프라의 주인공은 애플리케이션이 돌아가는 **컴퓨팅 자원**입니다.
가상 머신(VM)부터 컨테이너 오케스트레이션(K8s)까지 Terraform으로 관리합니다.

---

## 1. EC2 & Auto Scaling Group (ASG)

EC2 하나만 띄우는 건 쉽습니다. 하지만 서비스는 **ASG**로 띄워야 합니다.
서버가 죽으면 자동으로 살려내고, 트래픽이 늘면 늘려야 하니까요.

### Launch Template
"어떤 서버를 띄울지"에 대한 명세서입니다. (AMI, Instance Type, Security Group, User Data)

### User Data (Cloud-init)
서버가 뜰 때 실행할 스크립트입니다.
`templatefile` 함수를 써서 동적으로 스크립트를 생성할 수 있습니다.

```hcl
user_data = base64encode(templatefile("init.sh.tpl", {
  db_endpoint = var.db_endpoint
}))
```

---

## 2. Kubernetes (EKS/GKE) with Terraform

쿠버네티스 클러스터는 만드는 데 옵션이 100가지가 넘습니다.
직접 짜지 말고 커뮤니티 모듈을 쓰세요. 정신 건강에 이롭습니다.

### AWS EKS Module
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    green = {
      min_size     = 1
      max_size     = 3
      desired_size = 1
      instance_types = ["t3.medium"]
    }
  }
}
```
이 20줄 코드가 뒤에서는 500개의 리소스를 생성합니다. (VPC CNI, IAM Role, Node Group, Security Group...)

---

## 3. Terraform vs Helm vs Kubernetes YAML

*   **Terraform:** 인프라(클러스터 자체, 노드 그룹, IAM)를 만듭니다.
*   **Kubernetes YAML / Helm:** 클러스터 **안에** 들어가는 앱(Deployment, Service)을 만듭니다.

**경계선:** Terraform으로 Helm Provider를 써서 앱까지 배포할 수 있지만, 보통은 **ArgoCD** 같은 GitOps 도구에게 넘깁니다. Terraform은 **"클러스터 생성까지만"** 담당하는 게 국룰입니다.

---

## 🛠️ Lab: Launching an EKS Cluster

(비용 주의! 실습 후 바로 삭제하세요)

1.  Week 5의 VPC 모듈을 사용합니다.
2.  EKS 모듈을 호출하여 클러스터를 생성합니다.
3.  `aws eks update-kubeconfig` 명령어로 `kubectl` 접속 권한을 얻습니다.
4.  `kubectl get nodes`로 노드가 잘 떴는지 확인합니다.

---

## 📝 6주차 과제: Immutable Infrastructure

**목표:** 서버에 직접 들어가서 `apt-get update` 하지 않고, 이미지를 구워서 배포하는 **Packer + Terraform** 파이프라인을 설계하세요.

1.  **Packer:** Nginx가 설치된 AMI를 굽습니다.
2.  **Terraform:** `data "aws_ami"`로 최신 AMI ID를 조회합니다.
3.  **ASG:** 조회한 AMI로 Launch Template 버전을 업데이트합니다.
4.  **Instance Refresh:** ASG가 구버전 인스턴스를 죽이고 신버전으로 교체하는 설정을 추가하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** EC2 인스턴스가 처음 시작될 때 실행되는 스크립트를 전달하는 속성은? (user_data)
2.  **Q2.** Terraform에서 EKS 클러스터 내부의 리소스(Pod, Service 등)를 관리하기 위해 사용하는 Provider는? (kubernetes provider 또는 helm provider)
3.  **Q3.** Auto Scaling Group에서 인스턴스 사양이나 이미지를 변경했을 때, 기존 인스턴스를 순차적으로 종료하고 새 인스턴스로 교체해주는 기능은? (Instance Refresh)

---

이것으로 Part 2가 끝났습니다. 인프라는 다 만들어졌습니다.
Part 3에서는 이미 만들어진 인프라를 수정하고, 리팩토링하고, 테스트하는 **운영(Operations)**의 영역으로 들어갑니다.

**Cluster Ready.**
