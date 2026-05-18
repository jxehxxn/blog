---
layout: post
title: "Events Week 5: GitHub / GitLab Source"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## GitHub Source

```yaml
spec:
  github:
    mycorp-myapp:
      repositories:
        - owner: mycorp
          names: [myapp]
      webhook:
        endpoint: /push
        port: "12000"
      events: [push, pull_request, release]
      apiToken:
        name: github-token
        key: token
      webhookSecret:
        name: gh-webhook
        key: secret
      insecure: false
      active: true
```

webhook 자동 등록 (GitHub repo).

## Filter

```yaml
dependencies:
  - name: push
    eventSourceName: github
    eventName: mycorp-myapp
    filters:
      data:
        - path: body.ref
          type: string
          value: ["refs/heads/main"]
```

main branch만.

## GitLab

```yaml
spec:
  gitlab:
    mycorp:
      projects: ["mycorp/myapp"]
      events: [PushEvents, MergeRequestEvents]
      accessToken:
        name: gitlab-token
        key: token
```

거의 동일.

## Secret 검증

webhook secret으로 HMAC. 외부 위변조 차단.

## 자가평가
### Q1. webhook 자동 등록? **GitHub repo에 자동**. 정답 1.
### Q2. filter? **body.ref 기반**. 정답 1.
### Q3. secret 검증? **HMAC**. 정답 1.
### Q4. event type? **push/pull_request/release**. 정답 1.
### Q5. apiToken? **API 호출용 (repo info 조회)**. 정답 1.

## 다음
[Week 6: Kafka/Webhook].
