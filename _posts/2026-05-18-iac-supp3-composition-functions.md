---
layout: post
title: "IaC 보충 3: Crossplane Composition Functions — v1.14+ 혁명"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series supplement
---

7주차에서 짧게 다룬 Composition Functions를 깊게 풉니다.

## 1. 기존 Composition의 한계

```yaml
spec:
  resources:
    - name: rds
      base: ...
      patches:
        - fromFieldPath: spec.parameters.size
          toFieldPath: spec.forProvider.instanceClass
          transforms:
            - type: map
              map:
                small: db.t3.medium
```

- patch 문법이 제한적 (math, conditional, loop 어려움).
- 동적 자원 수 (예: replicas만큼 자원 N개) 불가.
- 외부 데이터 조회 불가.

## 2. Functions 도입

Pipeline 안에 함수(컨테이너) 여러 개를 chain. 각 함수가 자유로운 로직 구현.

```yaml
spec:
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef: { name: function-patch-and-transform }
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources: ...
    - step: go-templating
      functionRef: { name: function-go-templating }
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{- range $i := until (int .observed.composite.resource.spec.replicas) }}
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: Instance
            metadata: { name: "node-{{ $i }}" }
            spec: ...
            {{- end }}
    - step: validate
      functionRef: { name: function-cel }
      input: ...
```

각 step의 결과가 다음 step의 input.

## 3. 핵심 공식 Functions

### function-patch-and-transform
기존 patch 문법을 함수로 wrap. migration 시작점.

### function-go-templating
Go template로 manifest 생성. `range`, `if`, custom functions. 가장 유연.

### function-cel
CEL(Common Expression Language) 로 validate/transform. K8s admission과 같은 문법.

### function-extra-resources
다른 K8s 자원 조회 → composition 입력으로. 예: 같은 namespace의 ConfigMap 값 사용.

## 4. Custom Function 작성 (Go)

```go
type Function struct {
    fnv1beta1.UnimplementedFunctionRunnerServiceServer
}

func (f *Function) RunFunction(ctx context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
    // 1. composite (XR) 가져오기
    xr, err := request.GetObservedCompositeResource(req)
    if err != nil { return nil, err }

    // 2. 로직: 예) replicas만큼 instance 생성
    replicas := xr.Resource.GetInt64("spec.replicas")
    desired := response.To(req, response.DefaultTTL)
    for i := int64(0); i < replicas; i++ {
        instance := &unstructured.Unstructured{}
        instance.SetKind("Instance")
        instance.SetAPIVersion("ec2.aws.upbound.io/v1beta1")
        instance.SetName(fmt.Sprintf("node-%d", i))
        response.SetDesiredComposedResource(desired, fmt.Sprintf("node-%d", i), instance)
    }
    return desired, nil
}
```

빌드 후 컨테이너 이미지로 push:
```bash
crossplane xpkg build --package-root=. --package-file=function-myfn.xpkg
crossplane xpkg push xpkg.upbound.io/mycorp/function-myfn:v0.1.0
```

Function CR 설치:
```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata: { name: function-myfn }
spec:
  package: xpkg.upbound.io/mycorp/function-myfn:v0.1.0
```

## 5. Pipeline Pattern

전형적 pipeline:

```
1. patch-and-transform  (대부분의 단순 patch)
2. go-templating        (반복/조건)
3. extra-resources      (외부 K8s 데이터)
4. cel                  (validation)
5. auto-ready           (모든 자원 ready인지 종합)
```

각 step의 책임이 분리되어 추적 쉽다.

## 6. Migration 가이드

기존 patches → functions:
1. 같은 동작을 function-patch-and-transform으로 1:1 변환.
2. 동작 검증.
3. 복잡한 부분만 function-go-templating로 점진 이주.

## 7. 운영 함정

1. Function 이미지 cache 누락 → 매 reconcile마다 pull (느림).
2. Function timeout 짧음 → 큰 composition fail.
3. function 간 ordering 오류 → 의도와 다른 결과.
4. function 자체 메모리 부족 → OOM.
5. Custom function 버그가 사일런트 → cleanup 어려움.

## 8. 결론

Functions는 Composition을 사실상 미니 프로그램으로 변환한 혁명. 복잡한 사내 추상화에는 거의 필수. 단 운영 규율(테스트, 버전, 메트릭)이 더 중요해짐.
