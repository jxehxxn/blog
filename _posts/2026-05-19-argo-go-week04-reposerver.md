---
layout: post
title: "ArgoCD Go Week 4: repo-server"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## 책임
git clone + helm/kustomize render. gRPC API.

## RepoServerService
```go
service RepoServerService {
  rpc GenerateManifest(...) returns (ManifestResponse);
  rpc ListApps(...) returns (AppList);
  ...
}
```

## Cache
Redis로 result cache.

## CMP integration
sidecar process gRPC 호출.

## 자가평가
### Q1. 책임? **git + render**. 정답 1.
### Q2. API? **gRPC**. 정답 1.
### Q3. Cache? **Redis**. 정답 1.
### Q4. CMP? **sidecar gRPC**. 정답 1.
### Q5. 주요 메소드? **GenerateManifest**. 정답 1.
