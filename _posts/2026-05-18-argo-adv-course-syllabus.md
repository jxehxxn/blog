---
layout: post
title: "ArgoCD 심화 (Senior Series 8/19): Source Hydrator, CMP, Custom Generator"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced platform senior-series
---

## 강의 개요

ArgoCD 기본을 마친 시니어가 **ArgoCD 자체를 확장하고 깊게 운영**하기 위한 12주.

전제 비유: **ArgoCD가 LEGO 베이스라면, 심화는 자체 블럭 만들기**.

## 사전

- ArgoCD 기본 (Series 4 가정)
- Go (CMP, generator 작성)
- gRPC/REST API

## 학습 결과

- Source Hydrator (v2.13+) 운영.
- CMP plugin Go로 개발.
- ApplicationSet plugin generator 작성.
- ArgoCD API 직접 통합.

## 주차

| 주 | 주제 |
|----|------|
| 1 | 심화 overview |
| 2 | Source Hydrator |
| 3 | CMP 개발 |
| 4 | ApplicationSet plugin generator |
| 5 | Notifications customization |
| 6 | Resource hooks 깊게 |
| 7 | SSO advanced |
| 8 | ArgoCD Operator multi-tenancy |
| 9 | ApplicationSet generator 조합 패턴 |
| 10 | API 통합 (gRPC/REST/CLI) |
| 11 | Migration (Flux/Spinnaker → ArgoCD) |
| 12 | Capstone |

## 보충
1. Source Hydrator vs Image Updater
2. CMP best practices
3. ApplicationSet plugin SDK
4. API rate limit / pagination

## 다음

[Week 1].
