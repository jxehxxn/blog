---
layout: post
title: "CRD Week 2: kubebuilder init"
date: 2026-05-19 06:00:00 +0900
categories: crd kubebuilder senior-series
---

```bash
kubebuilder init --domain=mycorp.com --repo=github.com/mycorp/my-op
kubebuilder create api --group=db --version=v1 --kind=MyDatabase
```

자동 생성:
- api/v1/mydatabase_types.go
- controllers/mydatabase_controller.go
- config/crd/
- main.go

```bash
make manifests   # CRD YAML 생성
make install     # CRD 설치
make run         # 로컬 실행
```

## 자가평가
### Q1. kubebuilder? **operator scaffolding**. 정답 1.
### Q2. init? **프로젝트 초기화**. 정답 1.
### Q3. create api? **CRD + controller scaffold**. 정답 1.
### Q4. manifests? **CRD YAML 생성**. 정답 1.
### Q5. run? **로컬 controller 실행**. 정답 1.
