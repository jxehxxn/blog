---
layout: post
title: "Policy Week 9: Audit + Violations 관리"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno audit platform senior-series
---

## 학습 목표

- Background audit 운영.
- PolicyReport / ConstraintStatus 활용.
- Violations dashboard.
- Baseline 정리 전략.

## 1. 비유 — "정기 안전점검"

새 차량은 정문 검사. 운행 중인 차량도 정기 점검 — 도구 사용 후 새 규제 발효 시.

## 2. Gatekeeper Audit

기본 활성. 주기적 (60초) 모든 자원 평가.

```bash
kubectl get constraint
# NAME                  ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
# ns-must-have-owner    deny                 5
```

```bash
kubectl get constraint ns-must-have-owner -o yaml | yq '.status.violations'
```

## 3. Kyverno PolicyReport

```bash
kubectl get policyreport -A
# NAMESPACE   NAME         PASS   FAIL   WARN
# payments    pol-default  120    3      0
```

Pass/Fail 집계. fail 내역으로 위반 자원 식별.

## 4. Dashboard

Policy Reporter UI:
```bash
helm install policy-reporter policy-reporter/policy-reporter -n policy-reporter
```

Grafana dashboard로 violations 시각화.

## 5. Violations 분류

1. **알려진 baseline**: 정책 도입 전 만들어진 자원. 점진 수정.
2. **신규 위반**: 즉시 조사.
3. **외부 시스템**: 우리가 못 만지는 자원.

각자 다른 처리.

## 6. Exemption 관리

```yaml
spec:
  match:
    excludedNamespaces: [kube-system, istio-system]
  parameters: ...
```

또는 namespace label:
```yaml
match:
  - resources:
      kinds: [Pod]
      labelSelector:
        matchExpressions:
          - { key: policy-exempt, operator: DoesNotExist }
```

exempt 사유 + 만료 의무.

## 7. Notification

violations → Slack/PagerDuty.

```yaml
apiVersion: wgpolicyk8s.io/v1alpha1
kind: ClusterReportSink   # 가상 예시
spec:
  targets:
    - slack: { webhook: ... }
```

또는 policy-reporter의 내장 notification.

## 8. 메트릭

```promql
# Gatekeeper
gatekeeper_violations
gatekeeper_constraints

# Kyverno
kyverno_policy_results_total{status="fail"}
```

알람: violations 증가율 spike.

## 9. Compliance Report

분기 1회 자동 PDF:
- 정책 수.
- 위반 자원 수.
- 해결 시간 분포.
- exempt 사유.

SOC 2 evidence.

## 10. 자가평가 퀴즈

### Q1. Audit 가치?
1. **기존 자원 위반 발견 (admission만으론 부족)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. PolicyReport CR?
1. **wg-policy 표준 — pass/fail 집계**
2. UI 3. 무관 4. 임의 형식

**정답: 1.**

### Q3. Baseline 정리?
1. **점진, exempt + 수정 PR**
2. 모두 즉시 deny 3. UI 4. 무관

**정답: 1.**

### Q4. Exempt 운영 규율?
1. **사유 + 만료 의무**
2. 영구 3. UI 4. 무관

**정답: 1.**

### Q5. Compliance Report 가치?
1. **SOC 2 등 audit evidence**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음

[Week 10: External data + Sync].
