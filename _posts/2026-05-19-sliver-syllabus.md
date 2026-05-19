---
layout: post
title: "Sliver C2 깊이 알기: Senior 정보보안 연구가 12주 코스"
date: 2026-05-19 11:00:00 +0900
categories: security red-team blue-team c2 sliver senior
---

## 강의 개요

빅테크 / 보안 컨설팅 회사의 **Senior Offensive Security Engineer (Red Team) + Detection Engineer (Blue Team)** 양면 12주 과정.

**전제 — 법적·윤리적 사용**

- 모든 실습은 (1) 본인 소유 / (2) authorized engagement / (3) CTF / (4) 격리된 lab 환경에서만.
- 비인가 시스템 대상 사용은 **불법** (CFAA, 한국 정보통신망법). 절대 금지.
- 본 강의는 **red team 학습 + blue team 방어 패턴 이해**가 목표.
- 모든 챕터에 "blue team 관점 IoC + 탐지"를 동반.

비유: **무도장에서 호신술을 배우는 것**. 길거리에서 시비 걸 위해 X. 위협 패턴을 알아야 자기·고객을 지킬 수 있음.

## 사전 지식

- Linux/Windows internals (필수)
- Network 기초 (TCP/UDP/DNS/TLS) (필수)
- Go 기초 (custom extension)
- Pentest 기본 (Nmap, Metasploit 경험 권장)
- MITRE ATT&CK framework

## 학습 결과

12주 후:

1. Sliver server + client + implant 운영 (lab).
2. Listener 4종 (HTTPS, mTLS, DNS, WireGuard) 설계.
3. Beacon vs Session 모드 정확한 구분 + 사용.
4. BOF + custom extension 작성.
5. **방어자 관점**: Sliver 트래픽·implant IoC + EDR/NDR 탐지 설계.
6. Cobalt Strike / Mythic / Havoc과의 정확한 비교.
7. authorized red team engagement의 표준 절차.

## 주차 계획

| 주 | 주제 |
|----|------|
| 1 | 오리엔테이션 — C2 framework + 윤리/법 |
| 2 | 설치 + 첫 implant (lab) |
| 3 | Listener 종류 — HTTPS / mTLS / DNS / WireGuard |
| 4 | Implant 생성 옵션 + 포맷 |
| 5 | Session 모드 + Interactive |
| 6 | Beacon 모드 + Sleep/Jitter |
| 7 | Pivot / Port forward (lab network) |
| 8 | BOF (Beacon Object File) |
| 9 | Custom Extension 개발 (Go) |
| 10 | **Blue Team**: IoC + 탐지 패턴 |
| 11 | Sliver vs Cobalt Strike / Mythic / Havoc |
| 12 | Capstone — Authorized Red Team Engagement |

## 보충 포스트

- 보충 1: mTLS / Crypto 깊게
- 보충 2: DNS C2 동작 원리
- 보충 3: EDR Evasion — 방어자 관점
- 보충 4: MITRE ATT&CK 매핑

## 평가

- 자가평가 30%
- Lab 실습 30% (격리 환경)
- Capstone 40% (authorized engagement 시뮬레이션 + 방어 보고서)

## 다음 주차

[Week 1: 오리엔테이션]에서 C2 framework의 본질과 합법적 사용 범위, blue team 관점의 가치를 정리합니다.
