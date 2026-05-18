---
layout: post
title: "SRE Week 9: Capacity Planning"
date: 2026-05-19 07:00:00 +0900
categories: sre capacity senior-series
---

## 요소
- 현재 사용량.
- 성장률.
- peak 비율.
- buffer.

## 공식
needed = current × growth × peak × (1 + buffer)

## Modeling
- linear.
- exponential.
- seasonal (holiday).

## Headroom
보통 30~50% buffer.

## 자가평가
### Q1. 공식? **current × growth × peak × buffer**. 정답 1.
### Q2. peak? **trough 대비 비율**. 정답 1.
### Q3. buffer? **30~50%**. 정답 1.
### Q4. seasonal? **holiday 등 패턴**. 정답 1.
### Q5. 측정? **historical metric**. 정답 1.
