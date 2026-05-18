---
layout: post
title: "GoCD 깊이 알기: 12주 강의 계획서"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd thoughtworks pipeline
---

## 강의 개요

학부생~주니어가 **GoCD** (ThoughtWorks의 CI/CD 도구) 를 정복하는 12주 과정. Jenkins/GitHub Actions와 다른 GoCD만의 **"Continuous Delivery 책의 모범 구현"** 철학을 처음부터 풀어냅니다.

전제 비유: **GoCD = 자동차 조립 공장의 컨베이어 벨트 시스템**. 단순 빌드 트리거(Jenkins job)가 아니라, 각 부품(stage)이 어디서 어떻게 흘러가는지 한눈에 보이고(VSM, Value Stream Map), 어느 단계에서 사람 결재(manual approval)가 필요한지 명확합니다.

## 사전 지식

- Linux 셸, Git (필수)
- Docker 기초 (Elastic Agent 주차)
- Java 또는 일반 build tool 경험 (예시 이해용)
- YAML 읽기

## 학습 결과

12주 후:

1. GoCD server + agent production 배포.
2. Pipeline as Code (YAML)로 사내 CI/CD 운영.
3. Value Stream Map으로 multi-pipeline 흐름 시각화.
4. K8s/Docker Elastic Agent로 자동 scale.
5. Jenkins/GitHub Actions vs GoCD 정확한 차이 이해 + 의사결정.

## 주차 계획

| 주 | 주제 | 형식 |
|----|------|------|
| 1 | 오리엔테이션 — GoCD와 Continuous Delivery 철학 | 강의 |
| 2 | 설치 + 첫 Pipeline | 실습 |
| 3 | Pipeline / Stage / Job / Task 구조 | 실습 |
| 4 | Materials — Git, multi-material, polling | 실습 |
| 5 | Environment + Pipeline Group | 실습 |
| 6 | Pipeline as Code (YAML) | 실습 |
| 7 | Value Stream Map + Pipeline Dependency | 강의+실습 |
| 8 | Manual Approval + Trigger 종류 | 실습 |
| 9 | Elastic Agents (Docker, K8s) | 실습 |
| 10 | Plugins + Secret Management | 실습 |
| 11 | GoCD vs Jenkins/GHA/Tekton — 의사결정 | 강의 |
| 12 | 캡스톤 — 풀스택 CD 파이프라인 구축 | 프로젝트 |

## 보충 포스트

- 보충 1: Continuous Delivery 책 핵심 정리
- 보충 2: Value Stream Map 깊게
- 보충 3: K8s Elastic Agent 설정 가이드
- 보충 4: Jenkins → GoCD 마이그레이션

## 평가

- 자가평가 30%
- 실습 30%
- 캡스톤 40% (3종 모범답안)

## 다음 주차

[Week 1: 오리엔테이션]에서 GoCD의 출발점인 "Continuous Delivery" 책의 철학과, 다른 도구와 다른 GoCD의 차별점을 정리합니다.
