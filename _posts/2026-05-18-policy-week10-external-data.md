---
layout: post
title: "Policy Week 10: External Data + Sync — 다른 시스템과의 통합"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno platform senior-series
---

## 학습 목표

- Gatekeeper Sync로 cluster 내 다른 자원 참조.
- External Data Provider로 외부 시스템 통합.
- Kyverno context로 동적 데이터.
- CMDB / IAM 통합.

## 1. 비유 — "법정에서 증인 호출"

정책 판단에 다른 자원이 필요할 때. 같은 cluster 자원은 sync, 외부 시스템은 provider.

## 2. Gatekeeper Sync

```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata: { name: config, namespace: gatekeeper-system }
spec:
  sync:
    syncOnly:
      - { group: "", version: v1, kind: Namespace }
      - { group: "", version: v1, kind: ConfigMap }
```

Rego에서 접근:
```rego
ns := data.inventory.cluster.v1.Namespace[ns_name]
labels := ns.metadata.labels
```

## 3. External Data Provider

외부 HTTP service 호출.

```yaml
apiVersion: externaldata.gatekeeper.sh/v1beta1
kind: Provider
metadata: { name: cmdb-provider }
spec:
  url: https://cmdb-validator.mycorp.com/validate
  timeout: 5
  caBundle: ...
```

Rego:
```rego
response := external.send({
  "url": "cmdb-provider",
  "body": {"owner": labels.owner}
})
allowed := response.responses.status == "valid"
```

CMDB에 owner 라벨이 유효한지 확인.

## 4. Kyverno Context

```yaml
spec:
  rules:
    - name: ...
      context:
        - name: nsLabels
          apiCall:
            urlPath: "/api/v1/namespaces/{{request.namespace}}"
            jmesPath: "metadata.labels"
        - name: cmdbData
          apiCall:
            urlPath: "https://cmdb.mycorp.com/v1/owner/{{nsLabels.owner}}"
            method: GET
            jmesPath: "data"
      validate:
        deny:
          conditions:
            - key: "{{ cmdbData.active }}"
              operator: NotEquals
              value: true
```

K8s API + 외부 HTTP 모두.

## 5. 사용 사례

### CMDB 통합
namespace owner 라벨이 CMDB에 존재하는 active team 인지.

### IAM 통합
PodServiceAccount가 IAM에서 사용 가능 권한 검증.

### Budget 통합
새 자원 생성이 팀 예산 한도 안인지.

### Schedule 통합
"freeze 기간" 동안 변경 차단.

## 6. 성능

외부 호출은 admission latency 영향. 표준:
- timeout 5초.
- caching (Kyverno context cache).
- fallback (외부 down 시 어떻게).

## 7. 운영 함정

1. 외부 down 시 admission 자체 fail.
2. cache 미설정 → 외부 API 부담.
3. CA 검증 누락.
4. response schema 변경 → policy break.
5. circular dependency (admission → 외부 → API server).

## 8. 자가평가 퀴즈

### Q1. Sync의 가치?
1. **cluster 내 자원 Rego에서 참조**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. External Data Provider?
1. **외부 HTTP service 호출 (CMDB 등)**
2. K8s 내 3. UI 4. 무관

**정답: 1.**

### Q3. Kyverno context?
1. **K8s API + 외부 HTTP 둘 다 호출**
2. K8s 만 3. UI 4. 무관

**정답: 1.**

### Q4. 외부 호출 latency 대책?
1. **timeout + cache + fallback**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. circular dependency 위험?
1. **admission → 외부 → API server → admission → ...**
2. 안전 3. UI 4. 무관

**정답: 1.**

## 9. 다음

[Week 11: GitOps + lifecycle].
