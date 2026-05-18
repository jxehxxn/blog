---
layout: post
title: "Policy 보충 3: Policy Catalog 운영"
date: 2026-05-18 20:00:00 +0900
categories: policy platform senior-series supplement
---

빅테크 정책 카탈로그 관리.

## 1. 카탈로그 구조

```
policies/
  baseline/        # 모든 cluster 표준
    security/
    networking/
    cost/
  team-specific/
    payments/
    web/
  experimental/    # dryrun
```

## 2. 메타데이터

각 policy yaml에 annotation:
```yaml
metadata:
  annotations:
    policy.mycorp.com/owner: platform-team
    policy.mycorp.com/severity: high
    policy.mycorp.com/runbook: https://wiki.mycorp.com/...
    policy.mycorp.com/version: v2.1.0
```

## 3. 문서화

각 policy README:
- 무엇을 차단하는가.
- 왜 (rationale).
- exempt 절차.
- runbook.

## 4. Backstage Catalog

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata: { name: require-labels-policy }
spec:
  type: policy
  owner: platform-team
  lifecycle: production
```

사내 IDP에서 발견 가능.

## 5. 사용 통계

- 어떤 policy가 자주 trigger되는지.
- exempt 빈도.
- violation trend.

dashboard로 운영.

## 6. Versioning

policy도 SemVer. breaking change는 major + migration plan.

## 7. Test

각 policy unit test 필수.
```bash
opa test
kyverno test
```

CI에서 자동.

## 8. Review

분기 1회 policy review:
- 사용 안 되는 policy 제거.
- exempt 만료 갱신.
- 신규 도메인 추가.

## 9. 결론

Policy도 product. 카탈로그·문서·메트릭·리뷰 운영이 시니어 가치.
