---
layout: post
title: "Sliver Week 11: vs Cobalt Strike / Mythic / Havoc / Empire"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver comparison senior
---

## 학습 목표

- 5 C2 정량 비교.
- 시나리오별 선택.
- Migration 패턴.

## 1. 정량 비교

| 항목 | Cobalt Strike | Sliver | Mythic | Havoc | Empire |
|------|---------------|--------|--------|-------|--------|
| 라이선스 | 상용 $5,900/y | OSS Apache | OSS BSD | OSS GPL | OSS BSD |
| 언어 | Java/C | **Go** | Python/web | C++/Go | Python |
| UI | 강력 Java | CLI | Web | C++ GUI | CLI |
| Beacon Object File | **원조** | 지원 | 지원 | 지원 | X |
| Malleable profile | **있음** | 일부 | 있음 | 있음 | X |
| Aggressor scripting | 있음 | X | X | X | X |
| OS support | 광범위 | 광범위 | 광범위 | Win 중심 | Win/PS |
| Community | 거대 | 빠른 성장 | 활발 | 신규 | deprecated |

## 2. 시나리오별

### Mature red team + 예산 충분
**Cobalt Strike**. 표준, 자료 풍부.

### OSS + 학습/연구
**Sliver**. Go, 활발.

### Web UI + 모듈성
**Mythic**.

### Windows 전문
**Havoc** 또는 CS.

### Legacy PowerShell
Empire (deprecated, 권장 X).

## 3. 빅테크 / 컨설팅 사례

- Mandiant: CS 표준.
- BishopFox: Sliver (자체).
- 일부 modern red team: Mythic 또는 Sliver.

## 4. 위협 행위자 사용

연구·공개 reporting:
- Cobalt Strike: APT 다수.
- Sliver: 일부 ransomware affiliates (관측 증가).
- 위 사실은 blue team에 중요 (IoC + sigma rule 우선순위).

## 5. Migration

CS → Sliver 또는 반대:
- Listener 매핑.
- Profile 재작성.
- BOF 호환 (대부분).
- 팀 교육 필수.

## 6. 자가평가

### Q1. CS vs Sliver 라이선스?
1. **CS 상용, Sliver OSS**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q2. BOF 원조?
1. **Cobalt Strike**
2. Sliver 3. Mythic 4. Empire

**정답: 1.**

### Q3. Web UI?
1. **Mythic**
2. CS 3. Sliver 4. Havoc

**정답: 1.**

### Q4. Empire 상태?
1. **deprecated**
2. 표준 3. 신규 4. 무관

**정답: 1.**

### Q5. Mandiant 표준?
1. **Cobalt Strike**
2. Sliver 3. Mythic 4. Havoc

**정답: 1.**

## 7. 다음 주차

[Week 12: Capstone].
