---
layout: post
title: "IaC 보충 4: Terraform-Crossplane 하이브리드 — 두 도구가 공존하는 패턴"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series supplement
---

8주차에서 짧게 다룬 하이브리드를 자세히 풉니다.

## 1. 왜 둘을 같이 쓰는가

- 기존 Terraform 자산이 큼 (몇 년치 코드, 모듈).
- 새로 만드는 인프라는 Crossplane으로 가고 싶음 (GitOps + 추상화).
- 한 번에 마이그레이션은 위험.

빅테크 다수가 이 상황.

## 2. 표준 구도

```
[ Bottom: Terraform ]
- VPC, subnet, route table
- IAM root structure
- EKS cluster 자체
- Route53 zone
- KMS key
- 한 번 만들면 거의 안 바뀌는 기반

[ Middle: K8s cluster ]
- 위 Terraform이 만든 클러스터.
- 안에 Crossplane 설치.

[ Top: Crossplane ]
- 앱별 RDS, S3 bucket, IAM role.
- ProviderConfig는 Terraform이 만든 IAM 역할 참조.
- 자주 변경 + 셀프서비스 필요.
```

## 3. 경계 결정 기준

| 자원 | Terraform | Crossplane |
|------|-----------|------------|
| VPC, subnet, IGW | O | (Terraform이 만든 후 data source로) |
| IAM root role / OIDC | O | |
| EKS cluster | O | |
| Crossplane 자체 설치 | O (Helm 모듈) | |
| 앱별 RDS | | O |
| 앱별 S3 bucket | | O |
| 앱별 IAM role (IRSA) | | O |
| 사내 Tenant CR | | O |

## 4. State 이중 진실 — 위험

자칫 두 도구가 같은 자원을 manage하면 충돌. 절대 금지.

규율:
- 자원마다 "owner = terraform or crossplane" 라벨/tag 강제.
- OPA policy로 owner 미명시 거부.

## 5. 데이터 전달 — Terraform → Crossplane

Terraform output → SSM Parameter / ConfigMap에 적재 → Crossplane이 참조.

```hcl
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/mycorp/vpc/id"
  type  = "String"
  value = aws_vpc.main.id
}
```

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: SSMParameter
spec:
  forProvider:
    name: /mycorp/vpc/id
# Crossplane이 read해 dependency
```

또는 EnvironmentConfig (7주차) 에 값 적재.

## 6. Migration 점진 패턴

### Stage 1: 새 자원만 Crossplane
신규 마이크로서비스의 RDS는 Crossplane Claim으로. 기존은 그대로.

### Stage 2: refactor
같은 종류 자원 일부 Terraform → Crossplane 이전. `terraform state rm` + `crossplane Observe` 모드로 동시 인지 후 Terraform 측 코드 제거.

### Stage 3: 새 시스템 전체 Crossplane
3년 후쯤. 기존도 점진 이전.

## 7. 운영 도구 분리

- Atlantis: Terraform PR.
- ArgoCD: Crossplane CR sync.
- 같은 Git monorepo여도 path별 owner 다름.

## 8. Provider 인증 패턴

Crossplane ProviderConfig가 Terraform이 만든 IAM role을 참조:

```yaml
spec:
  credentials:
    source: IRSA   # ServiceAccount → IAM Role (Terraform이 만듦)
```

정적 키 없음. 안전.

## 9. 운영 함정 5선

1. 같은 자원을 두 도구가 manage → 충돌.
2. Terraform output을 SSM 등에 노출 안 함 → Crossplane이 hardcode.
3. ProviderConfig 자격증명 노출.
4. Migration 중 중간 상태가 너무 오래 (둘 다 못 만짐).
5. 사람마다 어디서 만들지 다름 (규율 부재).

## 10. 빅테크 사례

### Grafana Labs
새 인프라는 Crossplane, 기존은 Terraform. 5년 계획으로 점진 이주.

### Adobe
거의 모든 신규 자원 Crossplane. Terraform은 cluster 자체만.

### Netflix
Terraform 강고. Crossplane은 일부 셀프서비스 영역만.

## 11. 결론

하이브리드는 현실 답. 핵심은 **경계 명확 + owner 강제 + 점진 migration**. 시니어의 역할은 도구 선택보다 이 운영 규율 확립.
