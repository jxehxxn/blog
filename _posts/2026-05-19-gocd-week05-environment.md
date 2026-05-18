---
layout: post
title: "GoCD Week 5: Environment + Pipeline Group — 격리와 권한"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd
---

## 학습 목표

- Pipeline Group으로 namespace 분리.
- Environment로 agent + pipeline 묶기.
- RBAC 결합.

## 1. 비유 — "공장 안의 부서"

큰 공장(GoCD server) 안에 여러 부서(pipeline group):
- payments 부서.
- web 부서.
- mobile 부서.

각 부서가 자기 pipeline + agent 풀.

## 2. Pipeline Group

```yaml
groups:
  - group: payments
    pipelines:
      - payments-api
      - payments-batch
  - group: web
    pipelines:
      - web-frontend
      - web-cms
```

UI에서 그룹별 navigation. RBAC도 그룹 단위.

## 3. Environment

Environment = pipeline + agent + 변수 묶음.

```yaml
environments:
  - name: prod
    pipelines: [payments-api, web-frontend]
    agents: [agent-1, agent-2]
    environment_variables:
      AWS_REGION: us-east-1
      ENV: prod
```

같은 환경의 pipeline이 같은 agent + 같은 변수 사용. dev/stage/prod 분리.

## 4. RBAC

```yaml
admins: [admin1, admin2]
roles:
  - name: payments-admin
    users: [alice, bob]
  - name: payments-dev
    users: [carol]
```

Pipeline group에 role 할당:
```yaml
groups:
  - group: payments
    authorization:
      admins:
        roles: [payments-admin]
      operate:
        roles: [payments-dev]
      view:
        roles: [all-engineers]
```

권한:
- admin: 전체.
- operate: trigger / approve.
- view: 보기만.

## 5. Pipeline-level Authorization

```yaml
authorization:
  operate:
    roles: [prod-deployer]
```

stage manual approval에 role 지정 가능.

## 6. Tier 분리 패턴

빅테크:
- **Tier 0 group**: prod-critical. admin 1~2명만.
- **Tier 1 group**: prod. 팀 lead.
- **Tier 2 group**: stage. 개발자.
- **Tier 3 group**: dev. 모두.

## 7. 운영 함정 5선

1. Default group 사용 (없으면 모두 default) → 권한 부재.
2. Admin 너무 많음.
3. Environment 변수에 secret 평문.
4. Pipeline group rename → reference 깨짐.
5. Environment agent 할당 누락 → 빈 agent 풀.

## 8. 실습 과제

1. 2개 pipeline group 만들기 (payments, web).
2. 각 group에 다른 role 할당.
3. Environment dev/prod 분리 + 변수.
4. 사용자 A가 payments는 admin, web은 view만 되도록.
5. 잘못된 사용자 prod trigger 시도 → 거부 확인.

## 9. 자가평가 퀴즈

### Q1. Pipeline Group?
1. **namespace 분리 + RBAC 단위**
2. UI 색상 3. 빠른 빌드 4. 무관

**정답: 1.**

### Q2. Environment의 책임?
1. **pipeline + agent + 변수 묶음**
2. UI 3. 무관 4. 빌드 가속

**정답: 1.**

### Q3. RBAC 권한 3종?
1. **admins / operate / view**
2. read/write/exec 3. UI 4. 무관

**정답: 1.**

### Q4. operate?
1. **trigger + approve**
2. 보기 3. admin 4. 무관

**정답: 1.**

### Q5. Tier 0 group?
1. **prod-critical, admin 1~2명**
2. dev 3. 모두 4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 6: Pipeline as Code]에서 UI 클릭 → YAML 정의로 GitOps화.
