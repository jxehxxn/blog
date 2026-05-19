---
layout: post
title: "Sliver Week 12: Capstone — Authorized Red Team Engagement Simulation"
date: 2026-05-19 11:00:00 +0900
categories: security red-team blue-team c2 sliver capstone senior
---

## 캡스톤 개요

**Authorized 가상 engagement** 시뮬레이션. 자가 lab 또는 CTF 환경.

## 1. 시나리오 (Lab)

가상 회사 LabCorp:
- DMZ web server (target1).
- Internal AD domain (target2~5).
- 모두 자가 lab (Proxmox / VirtualBox).

목표:
1. 외부 access (web 취약점 — 자가 작성).
2. target1에 implant.
3. Pivot → 내부.
4. AD 탐색 → DA (Domain Admin) 또는 동등.
5. Loot 수집 (mock data).
6. **방어 보고서 작성**.

## 2. Red Team 산출물

1. **Engagement Plan** (Rules of Engagement, scope, dates).
2. **Timeline**: 각 phase 시각.
3. **Sliver setup**: listener, profile, OpSec.
4. **TTP 매핑**: MITRE ATT&CK 기법 ID.
5. **Loot manifest**.
6. **Cleanup**: 모든 implant + persistence 제거.

## 3. Blue Team 산출물

1. **IoC 목록**: 발견된 모든 신호.
2. **Detection rule**: YARA / Sigma / Suricata.
3. **Timeline reconstruction**: SIEM 기반.
4. **Gap analysis**: 못 잡은 phase + 개선.
5. **Recommendation**: 통제 강화.

## 4. 평가 (100점)

### 기준 1: Red Team (40)
- [10] OpSec (Beacon + jitter + working hour)
- [10] Pivot 성공 (2+ host)
- [10] TTP 다양성 (ATT&CK 5+ 기법)
- [10] Cleanup 완료

### 기준 2: Blue Team (40)
- [10] Static detection (YARA)
- [10] Network detection (Suricata/Zeek)
- [10] Behavioral (Sysmon/EDR sigma)
- [10] SIEM correlation

### 기준 3: 보고서 (20)
- [10] Red engagement report (executive summary)
- [10] Blue detection writeup + 개선 권고

## 5. 모범답안 옵션

### A. 단독 (red + blue 같은 사람)
purple team 학습. ~75.

### B. 팀 분리 (red + blue 별도)
실전 같음. ~90.

### C. CTF 출제
방어 + 공격 결합 시나리오. ~95.

## 6. 윤리·법 재확인

- 모든 활동은 lab 또는 본인 소유 환경.
- 외부 노출 X.
- 보고서에 ROE + 동의 명시.
- 보고서 외부 공유 시 민감 정보 제거.

## 7. 마무리

이 capstone을 마치면:
- **Red team**: authorized engagement 운영 가능.
- **Blue team**: 실제 C2 IoC 알고 detection 설계.
- **Purple team**: 두 영역 결합 → 조직 보안 개선.

빅테크 / 보안 컨설팅의 **Senior Offensive Security Engineer 또는 Detection Engineer** 즉시 가능.

## 8. 진로

- 빅테크 in-house red team (Google, Meta, MS).
- 보안 컨설팅 (Mandiant, BishopFox, NCC).
- Bug bounty researcher.
- Detection Engineering (Datadog, CrowdStrike).
- DFIR (Mandiant, Volexity).
- 학계 / R&D.
