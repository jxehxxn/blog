---
layout: post
title: "Sliver Week 1: 오리엔테이션 — C2 Framework + 법적/윤리적 사용"
date: 2026-05-19 11:00:00 +0900
categories: security red-team blue-team c2 sliver senior
---

## 학습 목표

- C2 framework의 정의와 역사.
- Sliver의 출범과 위치.
- **법적·윤리적 사용 범위** (가장 중요).
- Red team + Blue team 양면의 가치.

## 1. C2 = Command and Control

비유: 군 작전에서 본부(C2 server)가 야전 부대(implant/agent)에 명령 전달 + 보고 수신. Red team이 모의 침투 시 같은 모델로 가상 적군 행동을 시뮬레이션.

기술적으로:
- **C2 server**: operator가 명령 입력.
- **Implant (agent, beacon)**: target에 설치된 작은 program. server와 통신.
- **Listener**: server에서 implant 연결을 받는 endpoint.

## 2. 합법적 사용 범위 (필수 숙지)

C2는 **합법** 시나리오에서만 사용:
1. **본인 소유 시스템** (자기 노트북/VM).
2. **Authorized Penetration Test**: 서면 계약 (Rules of Engagement) 후.
3. **CTF**: HackTheBox, TryHackMe, Defcon CTF 등.
4. **격리 lab**: 인터넷 미연결 가상 네트워크.
5. **Blue team simulation**: 방어 훈련.

**불법** (절대 금지):
- 비인가 시스템 대상.
- 친구/회사 PC에 동의 없이.
- 데이터 절취/파괴.

한국 정보통신망법 + 미국 CFAA 위반 시 형사 처벌. **본 강의가 형사 책임을 면제하지 않음.**

## 3. Sliver 출범

- **BishopFox** (보안 컨설팅 회사) 출범 2020.
- Apache 2 OSS.
- Go 작성 → cross-platform.
- Cobalt Strike (상용 $5,900/year) 의 OSS 대안.

## 4. 다른 C2 비교

| 도구 | 라이선스 | 언어 | 특징 |
|------|---------|------|------|
| Cobalt Strike | 상용 (Fortra) | Java/C | 업계 표준, 유료 |
| **Sliver** | OSS Apache 2 | **Go** | 가장 빠르게 성장 |
| Mythic | OSS BSD | Python/web | 모듈형, web UI |
| Havoc | OSS GPL | C++/Go | modern, 새로움 |
| Empire | OSS BSD | Python | PowerShell 중심 (deprecated) |

빅테크 red team이 **Sliver 채택 증가** (OSS + Go + 활발한 community).

## 5. Red Team의 가치

조직 안에서:
- 실제 공격자 행동 시뮬레이션.
- 방어 통제(EDR/NDR/SIEM) 효과 검증.
- Incident response 훈련.
- Compliance (PCI-DSS 11.3 등).

방어팀(Blue team)의 능력 측정 = 공격팀의 가치.

## 6. Purple Team

Red + Blue 협업. 공격 시나리오 공유 → 방어 즉시 개선. 빅테크 표준.

## 7. 12주 로드맵

```
[기초] 1 오리엔테이션 → 2 설치 → 3 listener → 4 implant
[운영] 5 session → 6 beacon → 7 pivot
[심화] 8 BOF → 9 extension
[방어] 10 blue team IoC ⭐
[비교] 11 vs 다른 C2
[캡스톤] 12 authorized engagement
```

## 8. 자가평가 퀴즈

### Q1. C2 = ?
1. **Command and Control**
2. Cyber Crime 3. Cloud Computing 4. 무관

**정답: 1.**

### Q2. Sliver 모기업?
1. **BishopFox**
2. Google 3. MITRE 4. CrowdStrike

**정답: 1.**

### Q3. 합법적 사용에 포함되지 않는 것?
1. 본인 소유
2. Authorized pentest
3. CTF
4. **친구 PC에 몰래**

**정답: 4.** 동의 없는 사용은 불법.

### Q4. Sliver vs Cobalt Strike?
1. **Sliver OSS, CS 상용**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q5. Purple Team?
1. **Red + Blue 협업 → 방어 즉시 개선**
2. 다른 색 3. UI 4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 2: 설치 + 첫 implant]에서 격리 lab을 만들고 Sliver server를 띄웁니다.
