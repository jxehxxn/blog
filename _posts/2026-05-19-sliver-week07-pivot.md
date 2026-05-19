---
layout: post
title: "Sliver Week 7: Pivot / Port Forward (Lab Network)"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver senior
---

## 학습 목표

- Pivot 개념.
- portfwd / socks5 / SMB pivot.
- Lab network 시나리오.
- Blue team detection.

## 1. Pivot 정의

비유: 외부에서 회사 내부 IP에 직접 못 감. 첫 침해 호스트(beachhead)를 통해 내부로 이동.

기술적으로: 첫 implant가 proxy 역할. 다른 내부 호스트에 명령 전달.

## 2. Lab 시나리오

- attacker (외부).
- target1 (DMZ, beachhead, 외부 접근 가능).
- target2 (내부, target1만 접근).

목표: attacker → target1 → target2 (lab).

## 3. Port Forward

```sliver
sliver (TARGET1) > portfwd add --remote 192.168.10.5:22 --bind 127.0.0.1:2222
```

attacker의 127.0.0.1:2222 → target1을 통해 target2:22 (내부 SSH).

```bash
# attacker 머신
ssh user@127.0.0.1 -p 2222   # 사실은 target2로
```

## 4. SOCKS5 Proxy

```sliver
sliver (TARGET1) > socks5 start
```

attacker가 SOCKS5 proxy 활성 → 모든 도구가 target1 통해 내부 접근.

```bash
proxychains nmap 192.168.10.0/24
```

## 5. SMB Pivot (Windows)

target1 (Windows) → 내부 Windows host를 SMB named pipe로:

```sliver
sliver (TARGET1) > pivots smb --bind <bind-name>
sliver > generate --smb \\\\target1\\pipe\\... --save target2-implant
```

target2에 implant 실행 시 SMB로 target1 거쳐 attacker.

## 6. Lateral Movement (개념만)

도구:
- impacket (`psexec.py`, `wmiexec.py`, `smbexec.py`).
- CrackMapExec.
- Bloodhound (Active Directory 분석).

Sliver와 결합 → 내부 lateral.

## 7. Blue Team Detection

### Network
- 내부 호스트 간 비정상 트래픽 (Workstation → Workstation SMB).
- Lateral movement IoC: 4624 (login), 4672 (admin privilege), 4688 (process create).

### EDR
- Suspicious process chains.
- Pass-the-hash 시도.

### SIEM
- Bloodhound LDAP query 패턴.
- WinRM / PSExec usage anomaly.

## 8. Defense

- Network segmentation (microsegmentation).
- LAPS (local admin password).
- Tier model (workstation ≠ server ≠ domain controller).
- Just-in-time access.

## 9. 자가평가

### Q1. Pivot 정의?
1. **첫 침해 호스트를 통해 내부 이동**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. portfwd vs socks5?
1. **portfwd 단일 포트, socks5 모든 트래픽**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q3. SMB Pivot?
1. **named pipe로 Windows 내부 통신**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Lateral movement IoC 흔한?
1. **4624 / 4672 / 4688 + 비정상 host 간 SMB**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Defense 1?
1. **Network segmentation + Tier model**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 8: BOF].
