---
layout: post
title: "Ingress 보충 2: cert-manager + Vault PKI 깊게"
date: 2026-05-19 03:00:00 +0900
categories: cert-manager vault supplement
---

## 사내 PKI 통합
Vault PKI engine + cert-manager. 사내 도메인 cert 자동.

## CA chain
Root CA → Intermediate → Cert.
Root는 HSM에 격리, Vault는 Intermediate.

## 회전
짧은 cert (24~72h). 자동.

## 결론
사내 zero-trust + 자동화의 핵심.
