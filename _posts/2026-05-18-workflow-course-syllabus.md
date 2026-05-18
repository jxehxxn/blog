---
layout: post
title: "Argo Workflows + Tekton 깊이 알기 (Senior Series 3/19): 12주 강의 계획서"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 강의 개요

학부생~주니어가 **빅테크 Senior Platform Engineer 수준의 K8s-native CI/CD + Workflow Engine** 이해에 도달하기 위한 12주 과정. Argo Workflows와 Tekton을 모두 깊게 다루고, Jenkins/CircleCI 등 전통 CI와의 정확한 차이·이주 패턴까지 정리합니다.

전제 비유: **워크플로 엔진은 "공장 컨베이어 벨트의 OS"**. 한 제품을 만드는 데 필요한 (1) 부품 가공, (2) 조립, (3) 도장, (4) 검수, (5) 포장이 순서·병렬·재시도·조건으로 얽혀 있습니다. 이 흐름을 코드로 선언하고, 어디서 멈췄는지 추적하고, 재시작·재시도·관측·격리하는 것이 워크플로 엔진의 일.

## 사전 지식

- Kubernetes 기초 (필수, Course 1 추천)
- Docker / OCI image 기초
- Git, CI 개념 한 가지 이상
- YAML / JSON 읽기

## 학습 결과

12주 후:

1. Argo Workflows로 DAG 기반 복잡한 ML/데이터 파이프라인 운영.
2. Tekton으로 K8s-native CI 파이프라인 구성.
3. Argo Events로 외부 이벤트(웹훅, S3, Kafka) → workflow trigger.
4. Jenkins/CircleCI에서 K8s native로 이주.
5. Workflow의 보안(secrets, sandbox, supply chain) 운영.
6. 빅테크의 self-service workflow 플랫폼 설계.

## 주차 계획

| 주 | 주제 | 형식 |
|----|------|------|
| 1 | 오리엔테이션 — 왜 K8s 위에 워크플로 엔진 | 강의 |
| 2 | Argo Workflows 기초 — Workflow CR, templates | 실습 |
| 3 | Artifacts, parameters, input/output | 실습 |
| 4 | Workflow 패턴 — DAG, retry, suspend, exit handler | 실습 |
| 5 | Argo Events — sensor + eventsource | 실습 |
| 6 | Tekton 기초 — Task, Pipeline, PipelineRun | 실습 |
| 7 | Tekton triggers, resolvers, chains | 실습 |
| 8 | Argo Workflows vs Tekton — 의사결정 | 강의 |
| 9 | CI 패턴 — build/test/scan/sign/deploy | 실습 |
| 10 | GitOps 통합 — Workflow → ArgoCD | 실습 |
| 11 | 보안 — secrets, sandbox, supply chain | 강의+실습 |
| 12 | 캡스톤 — 풀스택 CI/CD 플랫폼 구축 | 프로젝트 |

## 보충 포스트

- 보충 1: Workflow 패턴 깊게 — DAG, fan-out/fan-in, MapReduce
- 보충 2: Tekton Chains + 공급망 보안
- 보충 3: Argo Workflows controller 성능 튜닝
- 보충 4: Jenkins → Argo/Tekton 마이그레이션 플레이북

## 다음 주차

[Week 1: 오리엔테이션]에서 Jenkins 시대의 한계와 K8s native 워크플로의 본질을 다룹니다.
