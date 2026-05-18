---
layout: post
title: "IaC Week 7: Crossplane Composition + XRD — 사내 인프라 패키지화"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- XRD(CompositeResourceDefinition)로 사내 추상화 CRD 만들기.
- Composition으로 XR → MR(여러 개) 매핑.
- Claim 모델로 namespace-scoped 사용자 인터페이스.
- v1.14+ Composition Functions 의 변혁.

## 1. 비유 — "Lego 세트 박스"

자동차를 만들 때 매번 바퀴·차체·창문 lego를 따로 사지 않습니다. "스포츠카 세트" 박스 하나가 모든 부품을 포함. **Composition = lego 세트, XRD = 세트의 박스 라벨, Claim = "이 세트 하나 주세요" 주문서.**

## 2. XRD — 추상화 정의

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.db.mycorp.com
spec:
  group: db.mycorp.com
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:
                      type: integer
                    version:
                      type: string
                      default: "16"
                    region:
                      type: string
```

- XR(`XPostgreSQLInstance`): cluster-scoped, infra 책임.
- Claim(`PostgreSQLInstance`): namespace-scoped, 앱팀 인터페이스.

## 3. Composition — 매핑 정의

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql.aws
spec:
  compositeTypeRef:
    apiVersion: db.mycorp.com/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            engine: postgres
            instanceClass: db.t3.medium
      patches:
        - fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - fromFieldPath: spec.parameters.version
          toFieldPath: spec.forProvider.engineVersion
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
    - name: backup-bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
      patches:
        - fromFieldPath: metadata.name
          toFieldPath: metadata.name
          transforms:
            - type: string
              string: { fmt: "%s-backups" }
```

XR 한 개로 RDS + 백업 S3가 함께 생성.

## 4. Claim — 앱팀 사용

```yaml
apiVersion: db.mycorp.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: payments-db
  namespace: payments
spec:
  parameters:
    storageGB: 100
    version: "16"
    region: us-east-1
  compositionSelector:
    matchLabels: { provider: aws }
  writeConnectionSecretToRef:
    name: payments-db-conn
```

`payments-db-conn` Secret이 자동 생성 (host, port, user, password).

## 5. 빅테크 패턴 — "Platform as a Product"

플랫폼팀 책임:
- XRD 정의 ("우리 회사가 제공하는 DB 인프라는 이런 모양")
- Composition 작성 (구현)
- ProviderConfig 관리
- RBAC, defaults, conventions

앱팀 책임:
- Claim 작성 (사용자 인터페이스만)

결과: **앱팀이 RDS/IAM/VPC를 몰라도 한 줄로 DB 프로비저닝**.

## 6. v1.14+ Composition Functions — 게임체인저

기존 Composition은 patch 문법이 제한적. 복잡한 로직 어려움.

Functions: Composition에 외부 컨테이너(Go/Python) 호출해 임의 로직.

```yaml
spec:
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef: { name: function-patch-and-transform }
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: rds
            base: ...
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.size
                toFieldPath: spec.forProvider.instanceClass
                transforms:
                  - type: map
                    map:
                      small: db.t3.medium
                      large: db.r5.xlarge
    - step: validate
      functionRef: { name: function-go-templating }
      input: { template: |
        # CEL/Go template로 추가 validate
      }
```

여러 function이 pipeline으로 연결. **Composition이 사실상 미니 프로그램**이 됨.

자세한 function 패턴은 [보충 3: Composition Functions 깊게].

## 7. EnvironmentConfig — 공유 변수

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata: { name: prod-env }
data:
  region: us-east-1
  vpcId: vpc-...
  defaultSubnet: subnet-...
```

Composition에서 참조해 모든 XR이 같은 환경 정보 사용.

## 8. 운영 함정 5선

1. XRD 변경 시 backward compatibility 깨질 위험 — version 정책 필수.
2. patches가 너무 많으면 추적 지옥. 5개 이상이면 Composition Function 권장.
3. providerConfigRef 누락 시 자원 생성 실패.
4. Composition labels 부실로 selector 의도와 다름.
5. backup/IAM 등 부가 자원 deletion policy를 Retain 안 둬서 실수 삭제.

## 9. 실습

```bash
# 1. XRD 정의 (XPostgreSQLInstance)
# 2. Composition AWS 버전 작성 (RDS + S3 backup)
# 3. Claim으로 payments-db 생성 → 콘솔 확인
# 4. Composition 변경 후 Claim 재생성 → 차이 비교
# 5. Composition Function 추가 (size → instanceClass mapping)
```

## 10. 자가평가 퀴즈

### Q1. XRD와 Claim 관계?
1. XRD = cluster-scoped 정의, **Claim = namespace-scoped 사용자 인터페이스**
2. 같은 것
3. Claim이 cluster scope
4. 무관

**정답: 1.**

### Q2. Composition의 핵심 가치?
1. **여러 MR을 묶어 사내 표준 인프라 패키지화**
2. 빠른 CLI
3. UI
4. 무관

**정답: 1.**

### Q3. v1.14+ Composition Functions 의 가치?
1. **외부 컨테이너로 임의 로직 — patch 한계 돌파**
2. UI
3. 무료
4. 무관

**정답: 1.**

### Q4. 앱팀이 한 줄 Claim으로 DB 프로비저닝하는 모델은?
1. **Platform as a Product — 플랫폼팀이 인프라, 앱팀이 사용자 인터페이스**
2. 무관
3. 비용
4. UI

**정답: 1.**

### Q5. patches 5개 이상의 권장?
1. **Composition Function으로 이전 — 추적성/가독성**
2. 그대로 둠
3. UI
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 8: Crossplane vs Terraform]에서 실제 의사결정 기준을 정리합니다.
