---
layout: post
title: "IaC 보충 2: Module 디자인 패턴 — Composition, Anti-patterns, Registry"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series supplement
---

4주차 모듈 디자인을 패턴·anti-pattern 수준으로 풉니다.

## 1. Good Module의 7가지 속성

1. **Single responsibility**: 한 모듈 한 역할.
2. **Stable interface**: input/output 자주 안 바꿈.
3. **Sensible defaults**: production-safe 기본값.
4. **Composable**: 다른 모듈과 결합 가능.
5. **Versioned**: SemVer, breaking은 MAJOR.
6. **Documented**: README + example.
7. **Tested**: terratest 등으로 e2e.

## 2. Module 디렉토리 표준

```
modules/vpc/
  README.md
  CHANGELOG.md
  main.tf
  variables.tf
  outputs.tf
  versions.tf
  examples/
    complete/
      main.tf
  test/
    vpc_test.go
```

## 3. Composition Pattern

모듈 안에서 다른 모듈 호출:

```hcl
module "vpc" {
  source = "../vpc"
  ...
}

module "eks" {
  source = "../eks"
  vpc_id = module.vpc.id
  ...
}
```

이게 자연스러우려면 input/output이 잘 정의되어 있어야.

## 4. Anti-patterns

### Anti-1: God Module
하나의 모듈이 VPC + EKS + RDS + IAM + S3 + Lambda 전부. 변경 영향 광범위, 재사용 어려움.

해결: 분리.

### Anti-2: Tight Coupling
모듈 A가 모듈 B의 내부 자원 직접 참조. B 변경 시 A 깨짐.

해결: B의 output만 사용, 내부는 unknown.

### Anti-3: Magic Defaults
default 값이 production에서 위험 (예: `encrypted = false`).

해결: secure-by-default.

### Anti-4: Version Range
`?ref=v1.x`. patch 자동 갱신 의도이나, 모듈 변경이 무방비 전파.

해결: 정확 tag (`?ref=v1.2.3`).

### Anti-5: Hidden Side Effects
모듈이 다른 리소스(KMS key, log group)를 만들면서 명시 안 함.

해결: README에 모든 created resource 명시.

## 5. Versioning 전략

### SemVer
- PATCH: bug fix, no input/output 변경.
- MINOR: 신규 input/output 추가 (backward compat).
- MAJOR: breaking change.

### Deprecation 절차
1. 새 input 추가 + 옛 input은 `deprecated` 마크.
2. minor release.
3. 다음 major에서 옛 input 삭제.
4. CHANGELOG에 마이그레이션 가이드.

## 6. Registry 종류

- **Public Terraform Registry**: registry.terraform.io.
- **사내 git**: `git::https://...//module?ref=v1.2.3`.
- **Terraform Cloud Private Registry**: 유료.
- **GitHub Releases**: tarball.

빅테크 표준: 사내 git monorepo + tag.

## 7. 사내 module monorepo

```
terraform-modules/
  vpc/
    v1.0.0 (tag)
    v1.1.0
  eks/
    v2.0.0
  rds/
    v0.1.0
```

각 모듈이 독립 tag. 한 repo에 다양한 모듈.

PR 시 변경된 모듈만 release. CHANGELOG 자동 생성.

## 8. Test — terratest

```go
func TestVPC(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../examples/complete",
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)
    vpcID := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

실제 클라우드에 띄워 검증 후 destroy. 비용 들지만 신뢰.

빅테크: 매 PR마다 사내 sandbox account에 terratest 자동.

## 9. Module Catalog

내부 wiki/Backstage에 모듈 카탈로그:
- 이름, 책임, 입력, 출력, 예시, owner.
- 검색 가능.
- 사용 cluster·repo 추적.

신규 엔지니어가 "이미 있는 모듈 또 만들기" 안 하도록.

## 10. 결론

Module은 IaC scaling의 핵심. 7대 속성 + anti-pattern 회피 + 강한 versioning + test가 빅테크 표준입니다.
