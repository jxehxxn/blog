---
layout: post
title: "IaC Week 3: Terraform State 깊게 — Backend, Locking, Workspaces, Import"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- State 파일이 무엇이고 왜 위험한지 안다.
- Remote backend(S3 + DynamoDB)로 협업 환경을 만든다.
- Workspaces vs 디렉토리 분리의 트레이드오프를 안다.
- 기존 자원을 state로 가져오는 `import` 사용법을 익힌다.

## 1. 비유 — "건축 도면 + 실측 기록부"

도면(코드)이 "이렇게 짓겠다"라면, State 파일은 "지금 실제로 이렇게 지어져 있다"의 기록부입니다.

Terraform이 plan을 만들 때 (1) 코드 (2) state (3) 클라우드 실제 상태를 비교해 차이를 계산. State가 없으면 매번 클라우드 전체를 스캔해야 하니 느리고, **리소스의 사용자 정의 메타데이터(예: terraform이 만든 random_id의 byte 값)** 같은 정보가 사라집니다.

## 2. State 파일의 구조

```bash
cat terraform.tfstate | jq '.resources | length'
# 42 리소스
```

내부는 JSON. 핵심:
- 각 리소스의 attribute (id, arn 등).
- random/computed 값.
- 의존성 그래프.
- **민감 정보** (DB 패스워드, 인증서 private key, IAM secret).

## 3. State의 위험

1. **평문 민감 정보**: state 파일이 노출되면 비밀 누출.
2. **동시 수정 충돌**: 두 사람이 동시 apply → state 깨짐.
3. **분실**: state 손실 → Terraform이 모든 리소스를 "처음 보는 것"으로 인지 → destroy/recreate 위험.
4. **버전 차이**: Terraform 새 버전이 옛 state 형식과 호환 안 될 수 있음.

해결: **Remote backend + lock + 백업 + access 제한**.

## 4. Remote Backend (S3 + DynamoDB)

```hcl
terraform {
  backend "s3" {
    bucket         = "mycorp-terraform-state"
    key            = "platform/network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

- S3: state 저장 + versioning(이전 버전 복원).
- DynamoDB: lock — 한 번에 한 사용자만 apply.
- KMS encryption.
- bucket policy로 access 제한 (write는 CI만, read는 팀원).

빅테크 표준.

### 권장 S3 설정
- versioning ON
- MFA Delete (실수 방지)
- SSE-KMS
- bucket public block 모두 ON
- access log 별도 bucket으로

## 5. State Lock 동작

```bash
# 사용자 A가 apply 시작
terraform apply
# Acquiring state lock. This may take a few moments...
# DynamoDB에 lock row 생성

# 사용자 B가 동시 apply 시도
terraform apply
# Error acquiring the state lock: ConditionalCheckFailedException
```

A가 끝나면 lock 해제. 만약 A의 프로세스가 죽어서 lock이 남으면:

```bash
terraform force-unlock <LOCK_ID>
```

**force-unlock 신중**. 진짜로 다른 사람이 apply 중인데 unlock하면 state 깨짐.

## 6. Workspaces

같은 코드로 dev/stage/prod 분리:

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform apply
```

내부적으로 state 파일 키가 `env:/prod/...` 형태로 분리.

### Workspaces vs 디렉토리 분리

| 항목 | Workspaces | 디렉토리 분리 |
|------|------------|---------------|
| 코드 중복 | 없음 | 약간 있음 (또는 모듈 공유) |
| 환경별 다른 provider 설정 | 어려움 | 쉬움 |
| state 격리 | 같은 backend, key 다름 | 완전 별도 |
| 잘못된 환경 apply 위험 | 높음 (workspace 헷갈림) | 낮음 |
| 빅테크 선호 | 적음 | **표준** |

빅테크는 거의 항상 **디렉토리 분리** (env/dev/, env/prod/). Workspaces는 임시 환경, ephemeral test에 적합.

## 7. Import — 콘솔로 만든 자원 끌어오기

기존 콘솔로 만든 EC2가 있다고 가정:

```hcl
# 코드 먼저 작성
resource "aws_instance" "legacy" {
  ami           = "ami-..."
  instance_type = "t3.micro"
  # ... 콘솔 값과 일치
}
```

```bash
terraform import aws_instance.legacy i-0abc123def
# state에 등록 (코드는 자동 생성 안 됨)

terraform plan
# 차이 0이 되도록 코드 다듬기 반복
```

빅테크에서는 **brownfield 마이그레이션(콘솔→IaC)**의 핵심 도구. Terraformer 같은 OSS는 import + 코드 자동 생성.

### import 블록 (v1.5+)
```hcl
import {
  to = aws_instance.legacy
  id = "i-0abc123def"
}
```

코드로 import 선언 → plan에 import + create 동시 표시.

## 8. State Surgery — `terraform state` 명령

```bash
terraform state list                              # 모든 자원
terraform state show aws_instance.web             # 상세
terraform state mv aws_instance.web aws_instance.app  # 이름 변경
terraform state rm aws_instance.legacy            # state에서만 제거 (실제 자원은 살아 있음)
terraform state pull > backup.tfstate              # 백업
terraform state push backup.tfstate                # 복원 (위험)
```

state 수술은 위험하지만 가끔 필요. 작업 전 항상 백업.

## 9. State 백업 정책

- S3 versioning ON (자동 백업).
- 매일 외부 S3 bucket으로 cross-region 복제.
- 분기 1회 백업에서 복원 drill.

## 10. 운영 함정 5선

1. **Local state로 협업**: 여러 사람이 충돌. remote backend 필수.
2. **State 파일 git commit**: 평문 비밀 노출.
3. **잘못된 workspace에 apply**: prod 변경이 dev에 (또는 반대).
4. **force-unlock 남발**: state 깨짐.
5. **versioning 미설정**: 잘못 변경 시 복원 불가.

## 11. 실습

```bash
# 1. S3 + DynamoDB로 backend 구성
# 2. 동료 1명과 동시 apply 시도 → lock 동작 확인
# 3. workspace로 dev/prod 분리 + 디렉토리 분리도 시도해 비교
# 4. 콘솔로 EC2 1개 만든 후 terraform import
# 5. terraform state mv 로 리소스 이름 리팩토링
```

## 12. 자가평가 퀴즈

### Q1. State 파일의 핵심 위험?
1. **평문 민감 정보 + 동시 수정 충돌**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q2. Remote backend의 필수 컴포넌트?
1. **저장소 (S3) + lock (DynamoDB)**
2. UI
3. CDN
4. 무관

**정답: 1.**

### Q3. Workspaces vs 디렉토리 분리 빅테크 표준?
1. workspaces
2. **디렉토리 분리**
3. 같음
4. 무관

**정답: 2.**

### Q4. import의 용도?
1. **콘솔로 만든 기존 자원을 state에 등록**
2. 새 자원 생성
3. state 백업
4. 무관

**정답: 1.**

### Q5. force-unlock의 위험?
1. **진짜 작업 중에 풀면 state 깨짐**
2. 안전
3. UI
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 4: Modules + Terragrunt]에서 코드 재사용·DRY 패턴을 다룹니다.
