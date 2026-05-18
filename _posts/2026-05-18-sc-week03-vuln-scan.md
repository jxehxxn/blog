---
layout: post
title: "SC Week 3: Vulnerability Scan — Trivy, Grype, Snyk"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain trivy platform senior-series
---

## 학습 목표

- CVE 데이터베이스의 본질.
- Trivy/Grype/Snyk 비교.
- CI 통합.
- False positive 관리.

## 1. CVE Database

- **NVD** (NIST): 공식.
- **GitHub Advisory Database**: 빠름.
- **OSV** (Google): aggregated.

각 scanner가 여러 source 통합.

## 2. Trivy

```bash
brew install trivy
trivy image mycorp/myapp:v1.2.3

# SARIF로 export (GitHub security)
trivy image --format sarif -o trivy.sarif mycorp/myapp:v1.2.3

# Source code scan
trivy fs .

# IaC scan
trivy config .
```

Aqua Security. OSS 표준.

## 3. Grype

Anchore. Syft와 같은 회사. SBOM 기반.

```bash
grype mycorp/myapp:v1.2.3
grype sbom:./sbom.json
```

## 4. Snyk

SaaS + CLI. 깊은 dep 분석.

```bash
snyk test
snyk container test mycorp/myapp
snyk monitor   # SaaS dashboard
```

## 5. CI 통합

```yaml
# GitHub Actions
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: mycorp/myapp:${{ github.sha }}
    exit-code: 1
    severity: HIGH,CRITICAL
```

CRITICAL/HIGH 발견 시 fail.

## 6. False Positive 관리

```yaml
# .trivyignore
CVE-2021-44228   # log4shell, 우리는 사용 안 함 (이유 명시)
```

또는 OPA로 정책:
```rego
deny[msg] {
  vuln := input.Results[_].Vulnerabilities[_]
  vuln.Severity == "CRITICAL"
  not in_allowlist(vuln.VulnerabilityID)
  msg := sprintf("CVE %v", [vuln.VulnerabilityID])
}
```

## 7. Severity 기준

- CRITICAL: 즉시 차단.
- HIGH: 다음 sprint.
- MEDIUM: 분기.
- LOW: backlog.

빅테크 SLA: CRITICAL은 24시간 내 patch.

## 8. Container vs Application

- Container scan: base image, OS 패키지.
- Application scan: 코드의 직접/간접 의존성.

둘 다 필요.

## 9. Vulnerability Disclosure Lag

CVE 발급 → DB update 까지 lag. zero-day 동안 scan 못 잡음.

대책: multi-source scanner + dynamic update.

## 10. 운영 함정

1. fail-fast (HIGH+ deploy 차단) 없음 → 위험 통과.
2. SBOM과 scan 분리 → 정보 손실.
3. False positive 미관리 → scan 무시.
4. Patch 적용 자동화 없음 → 분기 백로그만.
5. zero-day 대응 절차 부재.

## 11. 자가평가

### Q1. CVE Database 종류?
1. **NVD/GHSA/OSV** 2. UI 3. 무관 4. 1개

**정답: 1.**

### Q2. Trivy 강점?
1. **container/source/IaC 모두 scan**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. CI 차단 임계?
1. **CRITICAL/HIGH 보통**
2. ALL 3. UI 4. 무관

**정답: 1.**

### Q4. False positive 관리?
1. **명시적 ignore + 사유**
2. 무시 3. UI 4. 무관

**정답: 1.**

### Q5. CRITICAL SLA 빅테크?
1. **24시간 patch**
2. 1년 3. UI 4. 무관

**정답: 1.**

## 12. 다음

[Week 4: Cosign + Sigstore].
