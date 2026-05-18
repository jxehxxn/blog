---
layout: post
title: "ArgoCD 심화 Week 10: API 통합 — gRPC, REST, CLI"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced api platform senior-series
---

## 학습 목표

- ArgoCD API 3 종.
- Token 권한.
- Backstage 통합 예시.

## 1. API 종류

- **gRPC**: 가장 빠름. 사내 자동화.
- **REST**: 표준. 사용 가장 쉬움.
- **CLI**: argocd command.

## 2. REST 예시

```bash
ARGOCD=https://argocd.mycorp.com
TOKEN=$(argocd account generate-token --account ci --grpc-web)

curl -H "Authorization: Bearer $TOKEN" \
  $ARGOCD/api/v1/applications/myapp/sync \
  -X POST -d '{"prune": true}'
```

## 3. Application CRUD

```bash
# 생성
curl $ARGOCD/api/v1/applications -X POST -d @app.json

# 조회
curl $ARGOCD/api/v1/applications/myapp

# 수정
curl $ARGOCD/api/v1/applications/myapp -X PUT -d @app.json

# 삭제
curl $ARGOCD/api/v1/applications/myapp -X DELETE
```

## 4. Sync / Rollback

```bash
curl .../api/v1/applications/myapp/sync -X POST
curl .../api/v1/applications/myapp/rollback -X POST -d '{"id": 5}'
```

## 5. Backstage 통합

Backstage plugin이 ArgoCD API 호출:
- Service catalog에 "Deploy" 버튼.
- 클릭 → ArgoCD sync trigger.
- 사용자 자신의 JWT (SSO).

## 6. Token 권한 분리

- 사람 SSO token.
- CI Project JWT.
- API automation token (account.local 또는 SSO bot).

각각 권한 최소.

## 7. Rate Limit

ArgoCD 자체에 rate limit 없음 (기본). 사내 gateway/proxy로 추가.

## 8. Webhook

git → ArgoCD webhook으로 즉시 sync trigger.

```
https://argocd.mycorp.com/api/webhook
```

GitHub/GitLab/Bitbucket 자동 인지.

## 9. 운영 함정

1. token 권한 과다.
2. API 호출 폭주 → ArgoCD server 부담.
3. webhook URL 노출.
4. v1 vs v2 API 차이.

## 10. 자가평가

### Q1. API 3 종?
1. **gRPC + REST + CLI** 2. UI 3. 무관 4. 1개

**정답: 1.**

### Q2. Backstage 통합 의미?
1. **service catalog에서 deploy 버튼**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. Token 분리?
1. **사람/CI/automation 별 최소 권한**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Webhook 가치?
1. **git push 즉시 sync (polling 지연 제거)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Rate limit?
1. **사내 gateway로 보강**
2. ArgoCD 내장 3. UI 4. 무관

**정답: 1.**

## 11. 다음

[Week 11: Migration].
