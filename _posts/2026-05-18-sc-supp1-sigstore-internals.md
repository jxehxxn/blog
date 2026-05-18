---
layout: post
title: "SC 보충 1: Sigstore 내부 — Fulcio, Rekor 깊게"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain sigstore platform senior-series supplement
---

## Fulcio

- 단기 X.509 CA.
- OIDC token → 10분 cert.
- root CA: TUF (The Update Framework)로 분배.

내부:
1. Client → OIDC token + CSR.
2. Fulcio가 OIDC 검증 (Google/GitHub/Microsoft 등).
3. cert subject에 OIDC claim 박음.
4. cert 발급 (10분 유효).

## Rekor

- immutable append-only ledger.
- Trillian (Google CT 기반).
- 각 entry: signature + cert + artifact metadata.

inclusion proof: 특정 entry가 ledger에 포함됐다는 Merkle tree 증명.

## TUF

root metadata가 어떤 키를 신뢰할지 정의. Sigstore root metadata 5/5 인 threshold.

key compromise 시 root rotation.

## Private Sigstore

대규모 회사 self-hosting:
- Fulcio + private CA.
- Rekor private instance.
- 사내 OIDC.

## 결론

Sigstore는 단순 cosign 너머의 PKI ecosystem. 시니어는 내부 구조까지.
