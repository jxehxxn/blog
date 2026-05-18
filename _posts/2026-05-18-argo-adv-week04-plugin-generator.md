---
layout: post
title: "ArgoCD 심화 Week 4: ApplicationSet Plugin Generator"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced platform senior-series
---

## 학습 목표

- Plugin generator의 의미.
- HTTP service 구현.
- Use case (CMDB, IDP 통합).

## 1. Plugin Generator란

8 generator (List, Cluster, Git, Matrix, Merge, SCM, PR, Decision) 외 자체 정의.

```yaml
generators:
  - plugin:
      configMapRef: { name: my-plugin }
      input:
        parameters:
          query: "team=payments"
      requeueAfterSeconds: 60
```

ArgoCD가 plugin (HTTP service) 호출 → JSON 응답 받음 → template fan-out.

## 2. ConfigMap 설정

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: my-plugin, namespace: argocd }
data:
  token: "<bearer>"
  baseUrl: "https://my-plugin.svc.cluster.local"
```

## 3. HTTP Service 구현 (Go)

```go
func handler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ApplicationSetName string
        Input              struct {
            Parameters map[string]interface{}
        }
    }
    json.NewDecoder(r.Body).Decode(&req)

    // CMDB 조회 등
    teams := lookupTeams(req.Input.Parameters["query"])
    output := []map[string]interface{}{}
    for _, t := range teams {
        output = append(output, map[string]interface{}{
            "team": t.Name,
            "env":  t.Env,
        })
    }

    json.NewEncoder(w).Encode(map[string]interface{}{
        "output": map[string]interface{}{
            "parameters": output,
        },
    })
}
```

## 4. Use Case

- **CMDB 기반**: 팀 + 환경 자동.
- **IDP (Backstage)**: 등록된 service 자동.
- **Cloud API**: AWS account 목록 → cluster 자동 등록.
- **JIRA**: open ticket → preview env.

## 5. 보안

- bearer token.
- mTLS.
- private network only.

## 6. Best Practice

- timeout 합리적 (10s).
- caching (응답).
- retry policy.
- ApplicationSet 자체에 fallback.

## 7. 운영 함정

1. plugin down → ApplicationSet 멈춤.
2. output 형식 깨짐 → ArgoCD parse 실패.
3. token 노출.
4. requeue 너무 자주 → plugin 부담.

## 8. 자가평가

### Q1. Plugin generator 가치?
1. **자체 HTTP service로 사내 데이터 → Application** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. 응답 형식?
1. **JSON `output.parameters` 배열** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. 사용 사례?
1. **CMDB, IDP, Cloud API 통합** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. 보안?
1. **bearer + 사설 network** 2. 공개 3. UI 4. 무관

**정답: 1.**

### Q5. 운영 함정?
1. **plugin down 시 ApplicationSet 멈춤** 2. 안전 3. UI 4. 무관

**정답: 1.**

## 9. 다음

[Week 5: Notifications].
