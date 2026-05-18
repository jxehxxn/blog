---
layout: post
title: "IaC 보충 1: State 파일 깊게 — Lock Race, 복구, Secret 처리"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series supplement
---

3주차 state를 시니어 수준으로 풉니다.

## 1. State 파일 형식

JSON. 핵심 필드:
- `version`: state schema 버전.
- `terraform_version`: 만든 Terraform 버전.
- `serial`: 변경 카운터.
- `lineage`: state 파일 고유 UUID. 다른 state로 잘못 교체 방지.
- `outputs`, `resources`: 실제 데이터.

## 2. Lock의 정확한 동작 (DynamoDB)

DynamoDB table에 PK = `<bucket>/<key>` row 생성. 컬럼:
- `LockID`
- `Info`: 누가 언제 잡았는지
- `Operation`: plan or apply
- `Who`: user
- `Created`

해제 = row 삭제. 프로세스가 죽으면 row가 살아 남음 → force-unlock 필요.

빅테크 운영: Lock TTL 설정 (DynamoDB는 직접 지원 X, lambda 정리), Slack 알람 등.

## 3. Lock Race Condition

A와 B가 거의 동시 lock 획득 시도:
- DynamoDB `PutItem` ConditionExpression `attribute_not_exists(LockID)`로 atomic.
- A가 성공, B는 ConditionalCheckFailedException.

이 atomic 덕분에 race 없음. S3 자체는 race-prone(eventually consistent read였던 시절), 그래서 DynamoDB가 lock 전담.

(2020년 이후 S3 strong consistency라 일부 backend는 S3 만으로도 가능하나, DynamoDB 보호가 표준.)

## 4. State 복구 시나리오

### 시나리오 A: state 손실
- S3 versioning ON이면 이전 버전 복원.
- versioning OFF면 → `terraform import`로 한땀 한땀 복원.

### 시나리오 B: 잘못된 변경 commit
- backup state push: `terraform state push backup.tfstate`.
- 위험. 항상 백업 후.

### 시나리오 C: state 깨짐 (잘못된 force-unlock)
- terraform 명령 자체가 안 됨.
- backup에서 push.

빅테크: 분기 1회 state restore drill.

## 5. Sensitive 데이터

state에 평문 저장되는 것들:
- 자원의 모든 attribute (DB password, IAM secret access key 등).
- variable `sensitive=true`도 마찬가지.

대책:
1. **State backend encryption**: S3 KMS.
2. **State access 제한**: bucket policy로 platform 팀만.
3. **민감 secret은 state에 두지 않는다**:
   - DB 패스워드 → Vault external secret으로.
   - IAM 키 → IRSA로 정적 키 자체 안 만듦.

## 6. State 분할 전략

큰 monorepo는 plan 30분 + lock 충돌. 분할:

```
infra/
  global/    (route53, IAM 등 cross-region)
  per-region/
    us-east-1/
    eu-west-1/
  per-service/
    payments/
    auth/
```

각 디렉토리가 자체 state. 영향 범위 분리.

Terragrunt가 자연스럽게 이 패턴 강제.

## 7. State Surgery 예시

### 자원 이름 변경
```bash
terraform state mv aws_instance.old aws_instance.new
```

### 모듈 이동
```bash
terraform state mv module.web.aws_instance.this module.app.aws_instance.this
```

### State 분할
```bash
terraform state pull > full.tfstate
# 일부 자원만 새 state에 push
terraform state mv -state=full.tfstate -state-out=new.tfstate aws_xxx.foo aws_xxx.foo
```

### 자원 제거 (state만)
```bash
terraform state rm aws_legacy.foo
# 실제 자원은 살아 있음. 다른 도구로 관리할 때.
```

## 8. Drift된 state 정리

콘솔에서 자원 삭제됨, state에는 남음:
- `terraform apply` 시 자동 재생성 시도 (위험).
- 의도적이면 `terraform state rm` + 코드 삭제.

## 9. 운영 함정 모음

1. force-unlock 후 실제 작업 중이던 다른 사람의 state 손상.
2. state file 직접 vim 편집. 절대 금지.
3. backup 없이 state push.
4. sensitive variable이 plan 로그에 평문 출력 (CI가 sanitize 안 함).
5. 한 state에 너무 많은 자원 → plan 시간 폭증.

## 10. 결론

State는 Terraform 전체의 진실. 운영 규율(backup, lock, encryption, 분할)이 곧 시니어의 차이.
