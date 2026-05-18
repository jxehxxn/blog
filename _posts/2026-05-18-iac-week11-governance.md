---
layout: post
title: "IaC Week 11: Governance · Cost · Compliance — 정책·비용·감사를 코드로"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series compliance
---

## 학습 목표

- Policy as Code(OPA/Sentinel)로 사전 차단.
- Infracost로 PR 시점에 비용 가시화.
- SOC 2 / PCI-DSS 변경 통제와 IaC 매핑.
- Audit evidence를 자동 생성·보존.

## 1. 비유 — "건축 허가 + 비용 견적 + 감사"

도면이 있어도 (1) 건축법 위반(policy) 없어야 하고, (2) 예산 견적(cost) 있어야 하고, (3) 시공 후 감사(audit) 가능해야 합니다.

## 2. Policy as Code

### Open Policy Agent (OPA)
범용 정책 엔진. Terraform plan을 JSON으로 변환해 Rego로 검사.

```rego
package terraform.s3

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.after.block_public_acls == false
  msg := sprintf("Bucket %v has block_public_acls=false", [resource.address])
}
```

```bash
terraform show -json tfplan > plan.json
conftest test plan.json --policy policies/
```

CI에서 fail이면 머지 차단.

### HashiCorp Sentinel (유료)
Terraform Cloud/Enterprise에 통합. policy + audit 한 패키지.

### Crossplane용 OPA Gatekeeper
admission webhook으로 XR/Claim/MR 생성 시점 검사.

자세한 OPA/Kyverno는 [Tier 1 Course 6].

## 3. 자주 쓰는 policy

- S3 public access 차단.
- 태그 강제 (Environment, Owner, CostCenter).
- 인스턴스 타입 제한 (`>= xlarge` 거부).
- region 제한.
- IAM `*` 권한 거부.
- secrets manager에 KMS 없는 secret 거부.

## 4. Infracost — 비용 가시화

```bash
infracost breakdown --path .
# Total: $1,234.56/month
```

PR comment 통합:

```yaml
- uses: infracost/actions/setup@v3
  with: { api-key: ${{ secrets.INFRACOST_API_KEY }} }
- run: infracost diff --path . --format json --out-file infracost.json
- uses: infracost/actions/comment@v4
  with: { path: infracost.json }
```

PR마다 "이 변경이 월 $XXX 비용 변화" 표시. 큰 변경 시 PR 머지 전 결정 가능.

빅테크 표준: 큰 비용 변화(>$1000/월)는 cost owner 추가 승인 필수.

## 5. Tagging 강제

```hcl
# default_tags로 provider 레벨에서 강제
provider "aws" {
  default_tags {
    tags = {
      Environment = var.environment
      Owner       = var.owner
      CostCenter  = var.cost_center
      ManagedBy   = "terraform"
    }
  }
}
```

OPA로 tag 누락 검사. cost allocation 정확.

## 6. SOC 2 / PCI-DSS 매핑

### SOC 2 CC8.1 (Change Management)
- Authorization: GitHub branch protection + required reviewers.
- Documentation: PR description + plan output.
- Testing: terraform plan + OPA + Infracost.
- Approval: required code owners.
- Implementation: Atlantis apply.
- Audit: Git history + Atlantis log + S3 evidence.

### PCI-DSS 6.3 (Secure Development)
- IaC 자체가 secure development practice.
- Code review 필수.
- Sensitive 변경(IAM, security group)은 별도 reviewer.

### PCI-DSS 10 (Logging)
- 모든 apply 로그를 SIEM 적재.
- 1년 이상 보존 (PCI 요구), 빅테크 표준 7년.

## 7. Evidence Pipeline

```
[ PR ]
  ↓ Atlantis plan
[ plan output → S3 (WORM, 7년) ]
  ↓ merge
[ Atlantis apply ]
  ↓ apply log
[ S3 evidence ]
  ↓
[ Athena view: monthly change report ]
```

감사 시 1쿼리로 변경 이력·승인자·plan 결과 모두 회수.

## 8. 빅테크 운영 메트릭

- 월별 PR 수 / merge 수.
- OPA policy violation rate.
- Drift detection rate.
- Cost variance per PR.
- Audit evidence completeness.

이사회 보고용 dashboard.

## 9. 운영 함정 5선

1. **OPA policy CI 통과 후 apply 시점에 다른 결과**: plan은 변경되지 않았지만 cloud state가 바뀜. apply 시점에도 OPA 재실행 권장.
2. **Infracost 알람 없음**: 매월 $$$$ 청구서 보고 놀람.
3. **default_tags 누락**: cost allocation 부정확.
4. **Evidence 보존 짧음**: 감사 불통과.
5. **변경 승인이 단일 reviewer**: 사고 시 책임 분산 부족.

## 10. 실습

```bash
# 1. OPA conftest로 S3 public 차단 policy 작성
# 2. PR로 잘못된 plan 만들어 차단 확인
# 3. Infracost를 CI에 통합 → PR comment
# 4. 감사 SQL: 지난 30일 prod 변경 목록
# 5. default_tags 강제 + 누락 시 OPA fail
```

## 11. 자가평가 퀴즈

### Q1. Policy as Code 의 가치?
1. **사전 차단으로 잘못된 변경이 production에 안 도달**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q2. Infracost의 가치?
1. **PR 시점 비용 변화 가시화 → 의사결정 시점 제공**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q3. SOC 2 CC8.1 에 IaC가 매핑되는 핵심?
1. **PR이 곧 Change Request, Git history가 evidence**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. default_tags의 가치?
1. **모든 자원에 자동 태그 → cost allocation 정확**
2. UI
3. 비용 절감
4. 무관

**정답: 1.**

### Q5. evidence 보존 권장?
1. **7년 (가장 긴 규제 기준)**
2. 1일
3. 1주
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 12: 캡스톤]에서 모든 학습을 묶어 멀티클라우드 GitOps 인프라를 풀스택으로 구축합니다.
