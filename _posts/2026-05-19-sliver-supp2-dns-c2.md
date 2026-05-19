---
layout: post
title: "Sliver 보충 2: DNS C2 동작 원리"
date: 2026-05-19 11:00:00 +0900
categories: security c2 sliver dns supplement
---

## DNS C2 흐름

1. Implant → DNS query (TXT record): `<encoded-data>.attacker.com`.
2. attacker.com authoritative server (Sliver server)가 응답.
3. 응답에 명령/응답 encoded.

## Encoding

- base64 / base32.
- 짧은 length (DNS query label 63 char 한도).
- 큰 데이터는 여러 query 분할.

## Throughput

매우 느림. KB/s. 작은 명령 + 결과만.

## OpSec 강점

- DNS는 거의 모든 firewall이 허용.
- DoH (DNS over HTTPS)는 추적 더 어려움.

## Blue Team Detection

- Query 길이 분포 anomaly.
- 한 도메인에 query 폭주.
- TXT/A record 비정상 패턴.
- Sinkhole + IDS.

## DoH 위협

DoH 사용 implant → blue team이 DNS 자체 안 봄. HTTPS metadata만.

대응: SSL inspection + 사내 DNS 강제.

## 결론

DNS C2는 stealthy but slow. authorized lab 학습 후, blue team 입장에서 DNS 모니터링 강화.
