---
layout: post
title: "Sliver Week 3: Listener 종류 — HTTPS, mTLS, DNS, WireGuard"
date: 2026-05-19 11:00:00 +0900
categories: security red-team blue-team c2 sliver senior
---

## 학습 목표

- 4종 listener의 트레이드오프.
- 각자의 OpSec (operational security) 특성.
- Blue team 탐지 신호.

## 1. mTLS Listener (가장 강력)

```sliver
sliver > mtls --lhost 192.168.1.10 --lport 8888
```

- 양방향 TLS. server + client cert 검증.
- 외부 침해자가 implant 캡처해도 server에 연결 못 함.
- 단점: TLS handshake 패턴이 탐지 가능 (JA3).

### Blue team
- JA3/JA3S 핑거프린팅.
- mTLS는 정상 트래픽에서 드물어 anomaly.

## 2. HTTPS Listener

```sliver
sliver > https --lhost 192.168.1.10 --lport 443
```

- 정상 HTTPS로 위장.
- URL patterns, User-Agent 변형으로 정상 트래픽처럼.
- 단점: server 인증서 검증 약하면 MITM 가능.

### Blue team
- TLS SNI / cert subject 검사.
- 알려진 C2 hosting (자체 OSINT).
- URL pattern (정상 사이트 모방).

## 3. DNS Listener

```sliver
sliver > dns --domains attacker.com
```

- DNS TXT/A record로 데이터 전달.
- 극도로 느리지만 firewall 우회 가능.
- 단점: DNS query 패턴이 비정상.

### Blue team
- DNS query 길이 anomaly.
- 한 도메인에 query 폭주.
- TXT record 비정상.
- DNS over HTTPS (DoH)는 더 어려움.

## 4. WireGuard Listener

```sliver
sliver > wg --lhost 192.168.1.10
```

- VPN 같은 channel.
- 매우 안정.
- 단점: WG handshake 패턴.

### Blue team
- 알 수 없는 WG endpoint.
- UDP 51820 같은 표준 포트 외 다른 포트 사용.

## 5. Listener Selection Tree

| 환경 | 권장 |
|------|------|
| Lab 학습 | mTLS |
| Internal red team | mTLS / HTTPS |
| External (allowed engagement) | HTTPS + valid cert |
| Heavily monitored | DNS / WireGuard |
| Long-haul beacon | WG |

## 6. 다중 Listener

```sliver
sliver > mtls --lport 8443
sliver > https --lport 443
sliver > dns --domains backup.com
```

implant에 multiple C2:
```sliver
sliver > generate --mtls primary.com:8443 --https backup.com --dns dns.com
```

primary 실패 → fallback. resilience.

## 7. 트래픽 Profile (Malleable)

Cobalt Strike의 malleable profile처럼 Sliver도 HTTP request 커스터마이즈 가능 (config file).

OpSec: 정상 사이트 (Office 365, GitHub) 흉내.

## 8. 자가평가

### Q1. mTLS 강점?
1. **양방향 인증 + cert 검증**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. DNS C2 장단점?
1. **firewall 우회 / 매우 느림**
2. 둘 다 빠름 3. UI 4. 무관

**정답: 1.**

### Q3. WireGuard?
1. **VPN-like, 안정**
2. UDP만 3. UI 4. 무관

**정답: 1.**

### Q4. Multiple listener 가치?
1. **primary 실패 시 fallback**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Blue team JA3?
1. **TLS client 핑거프린팅**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 4: Implant 생성 옵션].
