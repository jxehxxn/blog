---
layout: post
title: "IaC Week 10: Drift Detection & Remediation — 코드와 현실의 차이 자동 감지"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- Drift의 4가지 발생 원인을 안다.
- Terraform/Crossplane 각각의 drift 정책을 안다.
- 자동 감지 cron + 자동 복구 정책을 설계한다.
- Drift evidence를 SIEM에 적재한다.

## 1. 비유 — "도면 vs 실측"

건축 후 시간이 지나면 도면과 실측에 차이가 생깁니다. (1) 인부가 임의 변경 (2) 자연 마모 (3) 다른 도면 적용 (4) 측정 오차.

인프라도 동일.

## 2. Drift의 4가지 원인

1. **사람의 콘솔 변경**: 가장 흔함. 긴급 변경 후 코드 미반영.
2. **자동 시스템의 변경**: HPA, Lambda 등이 자원 속성 변경.
3. **외부 자동화**: 다른 IaC 도구나 스크립트.
4. **클라우드 측 변경**: 드물지만 provider가 default 변경.

## 3. Terraform Drift 감지

```bash
terraform plan -detailed-exitcode
# exit 0: no changes
# exit 1: error
# exit 2: drift exists
```

cron으로 매 시간 또는 일별 실행 → exit 2면 알람.

```yaml
# GitHub Actions
name: drift-detect
on:
  schedule: [{cron: "0 * * * *"}]
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - id: plan
        run: terraform plan -detailed-exitcode -no-color || echo "exit=$?" >> $GITHUB_OUTPUT
      - if: steps.plan.outputs.exit == '2'
        run: |
          curl -X POST -H "Content-Type: application/json" \
            -d "{\"text\":\"Drift detected in ${{ github.repository }}\"}" \
            $SLACK_WEBHOOK
```

## 4. Spacelift / Atlantis의 drift detection

내장 기능으로 주기적 plan + 결과 dashboard. PR로 자동 fix 생성 옵션도.

## 5. Crossplane Drift 자동 복구

기본 동작: continuous reconcile. 콘솔에서 자원 변경 시 controller가 자동 복구.

### 정책 옵션
```yaml
spec:
  managementPolicies:
    - "*"   # observe + create + update + delete (기본)
    # 또는
    - "Observe"  # 변경 감지만, 복구 안 함
    - "Observe"
    - "Update"   # 일부 attribute만 관리
```

`Observe` 모드는 "현재 상태를 K8s status에 반영만, 변경 안 함". Migration 단계에서 유용.

## 6. Drift Evidence — SIEM 적재

빅테크는 모든 drift를 audit 대상으로:
- Terraform plan 결과를 S3 적재.
- Crossplane controller event를 fluent-bit으로 SIEM.
- 매 분기 audit 보고서에 drift 빈도·원인 분석.

```promql
# Crossplane drift event rate
sum(rate(crossplane_managed_unchanged_total[5m])) by (kind)
```

## 7. 자동 복구의 위험

자동 복구가 항상 옳지 않음:
- 운영 긴급 변경 (incident response) 중 콘솔로 임시 조치 → IaC 자동 복구가 사고 복구를 되돌릴 수 있음.

대책:
- 긴급 변경 시 reconcile 일시 정지: Crossplane `crossplane.io/paused: "true"` annotation.
- 또는 ProviderConfig 자체 비활성.
- 정상화 후 코드 반영 + reconcile 재개.

## 8. 흔한 패턴 — "Soft → Hard" 정책

도입 초기:
1. **Observe only**: 알람만, 복구 안 함. 사람이 PR로 코드/현실 일치.
2. 익숙해진 후 **자동 복구 ON**.
3. critical 자원만 별도로 manual 정책 유지.

## 9. False Positive 관리

자주 변하는 attribute는 ignore:

```hcl
resource "aws_autoscaling_group" "asg" {
  ...
  lifecycle {
    ignore_changes = [desired_capacity]   # ASG가 자동 조정
  }
}
```

Crossplane:
```yaml
spec:
  forProvider:
    desiredCapacity: 3
  ignoreFields:
    - spec.forProvider.desiredCapacity
```

## 10. 운영 함정 5선

1. **자동 복구만 두고 알람 없음**: 누구도 drift 발생을 모름.
2. **ignore_changes 남발**: 정작 중요한 변경 놓침.
3. **긴급 변경 후 reconcile 정지 안 함**: 자동 복구가 사고를 만듦.
4. **drift evidence 미보존**: audit에 사용 불가.
5. **cron 너무 자주**: AWS API rate limit.

## 11. 실습

```bash
# 1. Terraform: cron drift detection + Slack 알람
# 2. 콘솔로 S3 bucket policy 변경 → 1분 내 알람
# 3. Crossplane: 같은 자원에 자동 복구 확인
# 4. crossplane.io/paused annotation으로 일시 정지
# 5. Drift evidence를 S3에 적재
```

## 12. 자가평가 퀴즈

### Q1. Drift 가장 흔한 원인?
1. **콘솔 직접 변경**
2. cosmic ray
3. UI 버그
4. 무관

**정답: 1.**

### Q2. Terraform drift 감지 명령?
1. apply
2. **plan -detailed-exitcode (exit 2)**
3. destroy
4. import

**정답: 2.**

### Q3. Crossplane 기본 동작?
1. **Continuous reconcile + 자동 복구**
2. 감지만
3. 무관
4. 일회성

**정답: 1.**

### Q4. 긴급 변경 시 권장?
1. **reconcile 정지 후 콘솔 조치, 정상화 후 코드 반영 + 재개**
2. 그냥 콘솔 변경
3. IaC 무시
4. 무관

**정답: 1.**

### Q5. ignore_changes 남발의 위험?
1. **중요한 변경도 못 봄**
2. 안전
3. UI
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 11: Governance · Cost · Compliance]에서 IaC 정책·비용·감사를 다룹니다.
