---
layout: post
title: "Sliver Week 10: Blue Team — IoC + Detection 설계"
date: 2026-05-19 11:00:00 +0900
categories: security blue-team c2 sliver senior detection
---

## 학습 목표

- Sliver static IoC (YARA).
- Network detection (Suricata, Zeek).
- EDR behavioral (Sysmon, Falco).
- SIEM correlation.

## 1. 비유 — "지문 데이터베이스"

Sliver implant + 트래픽의 고유 패턴 → 탐지 rule.

## 2. Static (YARA)

```yara
rule Sliver_Implant {
    meta:
        author = "SOC"
        ref = "BishopFox/sliver"
    strings:
        $a = "BishopFox/sliver"
        $b = "wireguard.zx2c4.com"
        $c = "github.com/bishopfox/sliver"
        $d = { 73 6c 69 76 65 72 }  // "sliver"
    condition:
        2 of them
}
```

CrowdStrike, SentinelOne, ESET 등이 자체 rule. 사내 R&D도 추가.

## 3. Network (Suricata)

mTLS pattern:
```
alert tcp any any -> any any (msg:"Sliver mTLS pattern";
  tls.fingerprint; content:"<JA3>"; sid:1000001;)
```

JA3 핑거프린트 매칭. Sliver의 기본 JA3가 알려져 있음.

DNS pattern:
```
alert dns any any -> any any (msg:"Sliver DNS C2";
  dns.query; pcre:"/[a-z0-9]{30,}/"; sid:1000002;)
```

긴 random subdomain.

## 4. Zeek

zeek log → 트래픽 분석:
- ssl.log: cert subject anomaly.
- conn.log: 정기 callback 패턴.
- dns.log: query 길이 분포.

## 5. Sysmon (Windows)

Sysmon event ID 1 (process create), 3 (network connect), 7 (image load), 11 (file create).

Sliver 탐지 sigma rule:
```yaml
title: Sliver Beacon Network Pattern
detection:
  selection:
    EventID: 3
    Image: 'C:\Users\*\AppData\*'
    DestinationPort: [8443, 443]
  filter:
    ParentImage|endswith: ['svchost.exe', 'explorer.exe']
  condition: selection and not filter
```

## 6. Falco (Linux)

Linux behavioral:
```yaml
- rule: Suspicious Outbound C2
  desc: implant 의심 outbound
  condition: >
    outbound and not (fd.sport in (80, 443))
    and proc.exepath startswith /tmp
  output: "Suspicious outbound (cmd=%proc.cmdline)"
```

## 7. SIEM Correlation

여러 신호 결합:
- Suricata: JA3 match.
- Sysmon: process create 이상.
- DNS: 긴 query.

= confidence "Sliver implant 가능성 95%".

Splunk ES / Sentinel correlation rule.

## 8. Threat Intel

C2 server IP/도메인 IoC feed:
- Abuse.ch ThreatFox.
- MalwareBazaar.
- 사내 Threat Intel.

SOAR 자동 차단.

## 9. Detection 검증

Red team이 의도적 Sliver 실행 → Blue team이 잡는가 측정. **Purple team exercise**.

KPI:
- Detection rate.
- Time to detect (MTTD).
- False positive rate.

## 10. 자가평가

### Q1. YARA?
1. **static binary 패턴 rule**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. JA3?
1. **TLS client 핑거프린트**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. Sysmon event 3?
1. **network connect**
2. process 3. file 4. registry

**정답: 1.**

### Q4. Falco?
1. **Linux runtime detection**
2. Windows 3. UI 4. 무관

**정답: 1.**

### Q5. Purple team?
1. **Red 실행 + Blue 탐지 측정**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 11: 다른 C2 비교].
