---
layout: post
title: "Sliver 보충 1: mTLS / Crypto 깊게"
date: 2026-05-19 11:00:00 +0900
categories: security c2 sliver crypto supplement
---

## Sliver mTLS

- Per-implant cert.
- Server CA가 서명.
- Implant도 client cert.

## Handshake

1. TCP connect.
2. TLS ClientHello (with cert).
3. Server ClientHello + cert + CertificateRequest.
4. Client cert verify.
5. Symmetric session.

## JA3 핑거프린트

Sliver 기본 JA3가 알려져 있음 → blue team rule.

OpSec: utls 같은 라이브러리로 Chrome/Firefox JA3 흉내. Sliver는 일부 지원.

## Cert 관리

- Server CA: ~/.sliver/configs/server/.
- Operator cert: 분배 신중.
- Implant cert: per-build.

CA key 노출 = 모든 통제 무력.

## 결론

mTLS가 가장 강력하지만 JA3 detect 가능. 추가 위장 layer 필요.
