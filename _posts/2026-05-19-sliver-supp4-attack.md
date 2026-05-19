---
layout: post
title: "Sliver 보충 4: MITRE ATT&CK 매핑"
date: 2026-05-19 11:00:00 +0900
categories: security c2 sliver attack supplement
---

## MITRE ATT&CK

공격자 행동 분류 framework. Tactic > Technique > Sub-technique.

## Sliver 행동 매핑

| Sliver 기능 | ATT&CK |
|------------|--------|
| Initial implant | T1059 Command-Line / T1204 User Execution |
| mTLS C2 | T1573.002 Encrypted Channel: Asymmetric Crypto |
| HTTPS C2 | T1071.001 Application Layer Protocol: Web |
| DNS C2 | T1071.004 DNS |
| Registry persistence | T1547.001 Registry Run Keys |
| Service persistence | T1543.003 Windows Service |
| Scheduled task | T1053.005 Scheduled Task |
| Process injection | T1055 Process Injection |
| Port forward | T1572 Protocol Tunneling |
| SOCKS proxy | T1090.001 Internal Proxy |
| SMB pivot | T1021.002 Remote Services: SMB |
| Credential dump (via BOF) | T1003 OS Credential Dumping |
| Discovery (ls, ps) | T1083 File Discovery / T1057 Process Discovery |

## 방어 매핑

각 ATT&CK 기법 → 방어 control:
- T1071: TLS inspection + JA3.
- T1547: registry monitoring (Sysmon 13).
- T1055: ETW + EDR.
- T1003: LSASS protection.

## 활용

- Purple team exercise.
- Coverage gap analysis.
- Detection KPI.
- Red team 보고서.

## ATT&CK Navigator

MITRE 공식 도구. JSON으로 매핑 시각화.

## 결론

ATT&CK은 공격·방어 공통 언어. 모든 engagement 보고서가 ATT&CK ID로 작성됨.
