---
layout: post
title: "SC Week 11: 공급망 Incident Response"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain incident platform senior-series
---

## 학습 목표

- 4 종류 공급망 사고 대응.
- 영향 범위 분석.
- Communication.

## 1. 사고 유형

1. **CVE in dep**: Log4Shell 같은 사고.
2. **Compromised key**: cosign 서명 키 유출.
3. **Compromised dep**: 의존성 패키지 악성 변경.
4. **Insider/compromised CI**: 빌드 시스템 침해.

## 2. CVE in dep 대응

1. SBOM database 검색 → 영향 image 식별.
2. 영향 namespace/cluster 매트릭스.
3. Patch 또는 mitigation.
4. emergency rebuild + redeploy.
5. Post-mortem.

Log4Shell case: 24시간 내 영향 식별 + 일주일 안에 모든 patch.

## 3. Compromised Key 대응

1. Key 즉시 revoke.
2. Rekor로 의심 서명 시각 검색.
3. 영향 image 재서명.
4. Admission policy 강화 (특정 issuer 차단).
5. Post-mortem + 키 운영 개선.

Keyless 사용 시 더 쉬움 (개별 cert 만료 10분).

## 4. Compromised Dep 대응

가장 어려움. 어떤 image가 영향인가?

1. SBOM 검색.
2. 영향 image 모두 quarantine.
3. dep 교체 + rebuild + redeploy.
4. 운영 monitoring 강화.

xz utils case: 거의 모든 Linux distro 영향. 다행히 stable 전 발견.

## 5. Compromised CI

1. CI 시스템 격리.
2. 모든 최근 build artifact 재검증 (Rekor + provenance).
3. 의심 image 재빌드 (clean environment).
4. CI 보안 강화.

## 6. Communication

- 내부: incident channel + 시간순 update.
- 외부: 고객 영향 시 statuspage.
- 정부: 일부 사고는 보고 의무 (CIRCIA 2024).

## 7. Tabletop Exercise

분기 1회 가상 시나리오:
- "오늘 OpenSSL CVE-X가 발표됐다. 30분 안에 영향 평가."

훈련된 팀이 진짜 사고에서 빠름.

## 8. 운영 함정

1. SBOM 없음 → 영향 분석 불가.
2. emergency rebuild 절차 미정의.
3. Rekor 미사용 → 의심 서명 추적 불가.
4. Tabletop 미실행.
5. 외부 communication 절차 부재.

## 9. 자가평가

### Q1. CVE in dep 1차 대응?
1. **SBOM 검색으로 영향 image 식별**
2. 무시 3. UI 4. 무관

**정답: 1.**

### Q2. Compromised key + Rekor?
1. **시각으로 의심 서명 검색**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. Keyless 장점 (key 사고)?
1. **개별 cert 10분 만료 → 영향 범위 자동 제한**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Tabletop 가치?
1. **훈련 → 실전 빠름**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. CIRCIA 2024?
1. **공급망 사고 정부 보고 의무 (US)**
2. UI 3. 무관 4. 가격

**정답: 1.**

## 10. 다음

[Week 12: 캡스톤].
