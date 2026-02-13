---
layout: post
title: "Infrastructure as Code with Terraform: Week 12 - Capstone: Enterprise Grade Infrastructure Deployment"
---

축하합니다. 12주의 여정을 마쳤습니다.
HCL 문법도 모르던 여러분이 이제는 정책 코드로 보안까지 제어하는 전문가가 되었습니다.

마지막 과제는 여러분이 배운 모든 지식을 쏟아부어, 실제 유니콘 기업 수준의 인프라를 처음부터 끝까지 구축하는 것입니다.

---

## 🏆 Capstone Project: "Global E-Commerce Platform"

**시나리오:**
전 세계에 서비스를 제공하는 쇼핑몰 "TerraShop"의 인프라를 구축하세요.

### 1. Requirements

**Architecture:**
*   **Multi-Environment:** Dev, Stage, Prod 환경 분리 (폴더 구조).
*   **Network:** VPC (Public/Private/Database Subnets), NAT Gateway, Bastion Host.
*   **Compute:** EKS Cluster (Managed Node Group).
*   **Database:** RDS Aurora (Multi-AZ).
*   **Storage:** S3 (Static Web Hosting + CDN CloudFront).

**Operations:**
*   **Remote Backend:** S3 + DynamoDB Locking.
*   **CI/CD:** GitHub Actions로 PR 시 Plan, Merge 시 Apply 자동화.
*   **Modules:** VPC, EKS, RDS는 반드시 모듈화하여 재사용할 것.

**Security:**
*   **IAM:** 최소 권한 원칙 적용. (개발자는 ReadOnly)
*   **Compliance:** OPA/Checkov를 통해 "S3 Public Open 금지" 정책 적용.

### 2. Deliverables (제출물)

1.  **GitHub Repository:** 모든 Terraform 코드.
2.  **Architecture Diagram:** Draw.io 등으로 그린 구성도.
3.  **Demo Video:**
    *   `terraform apply`로 인프라가 생성되는 과정.
    *   생성된 EKS 클러스터에 Nginx 파드를 띄우고 접속하는 모습.
    *   PR을 올렸을 때 CI 봇이 Plan 결과를 댓글로 다는 모습.

---

## 3. The "Day 2" Checklist

인프라를 만드는 건 "Day 1"입니다. 운영하는 "Day 2"가 더 중요합니다.

*   [ ] **Tagging Strategy:** 비용 정산을 위해 모든 리소스에 `Owner`, `CostCenter`, `Environment` 태그가 붙어있는가?
*   [ ] **Backup:** RDS와 S3의 백업 정책이 설정되어 있는가?
*   [ ] **Cost Estimation:** `infracost` 같은 도구로 예상 비용을 확인했는가?
*   [ ] **Disaster Recovery:** 리전 전체가 날아가면 다른 리전에 띄울 수 있는가? (IaC라면 가능해야 함)

---

## 🎓 Closing Remarks

인프라를 코드로 관리한다는 것은 단순히 "자동화"를 의미하는 게 아닙니다.
**"인프라를 신뢰할 수 있게 만드는 것(Reliability)"**입니다.

새벽 3시에 장애가 터져도, 손을 떨며 콘솔을 클릭하는 대신 침착하게 코드를 수정하고 PR을 올릴 수 있는 자신감. 그것이 여러분이 얻어가는 가장 큰 자산입니다.

여러분의 코드가 세상을 지탱하는 단단한 기반이 되기를 바랍니다.
수고하셨습니다.

**Terraform apply complete! Resources: 100 added, 0 changed, 0 destroyed.**
