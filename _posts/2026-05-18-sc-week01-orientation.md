---
layout: post
title: "SC Week 1: 오리엔테이션 — SolarWinds, Log4Shell, xz utils"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain platform senior-series
---

## 학습 목표

- 공급망 공격의 4 유형.
- 실제 사례 3건.
- SLSA framework 소개.
- US EO 14028 영향.

## 1. 4 유형

1. **Source 공격**: 코드 자체 삽입 (SolarWinds).
2. **Build 공격**: 빌드 시스템 탈취.
3. **Dependency 공격**: 의존성 패키지 삽입 (typo, takeover).
4. **Distribution 공격**: 배포 채널 (CDN, registry) 탈취.

## 2. 사례 1: SolarWinds (2020)

빌드 시스템에 SUNBURST 백도어. 19,000+ 고객 배포. US Treasury, Microsoft, FireEye 포함. 영향 수년.

교훈: 빌드 시스템 자체가 supply chain target.

## 3. 사례 2: Log4Shell (2021)

log4j 2.x 의 JNDI lookup → RCE. Java 거의 모든 enterprise 영향. 며칠간 대규모 패닉.

교훈: 의존성의 의존성(transitive)까지 SBOM으로 추적해야.

## 4. 사례 3: xz utils (2024)

Linux 핵심 압축 라이브러리 maintainer가 사회공학으로 백도어 삽입. Debian/Ubuntu unstable 배포 직전 우연히 발견.

교훈: OSS maintainer 1인 의존성도 supply chain 위험.

## 5. SLSA Framework

Google 주도. 4 레벨로 supply chain 성숙도 측정.

- L1: Provenance 존재.
- L2: 빌드 서비스 사용.
- L3: hosted build + 서명된 provenance.
- L4: 2인 리뷰 + hermetic + reproducible.

빅테크 표준: L3 최소.

## 6. US Executive Order 14028 (2021)

미국 연방 SW 공급에 SBOM 의무. NIST SP 800-218 (SSDF). 미국 시장 빅테크는 모두 영향.

## 7. 12주 로드맵

```
[검증] 1 → 2 SBOM → 3 scan → 4 cosign → 5 SLSA → 6 in-toto
[운영] 7 Tekton Chains → 8 admission → 9 풀스택
[준수] 10 compliance → 11 incident → 12 캡스톤
```

## 8. 자가평가

### Q1. 공급망 공격 4 유형?
1. **Source/Build/Dependency/Distribution** 2. UI 3. 무관 4. CPU

**정답: 1.**

### Q2. SolarWinds 교훈?
1. **빌드 시스템도 target** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q3. SLSA 단계?
1. **L1~L4** 2. 1~3 3. UI 4. 무관

**정답: 1.**

### Q4. EO 14028의 핵심?
1. **연방 SW SBOM 의무** 2. UI 3. 무관 4. 가격

**정답: 1.**

### Q5. xz utils 교훈?
1. **OSS maintainer 1인 의존도 위험**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음

[Week 2: SBOM].
