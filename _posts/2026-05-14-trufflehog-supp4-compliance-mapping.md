---
layout: post
title: "TruffleHog 보충 4: 컴플라이언스 매핑 — SOC 2, PCI-DSS, HIPAA, ISO 27001"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops compliance supplement
---

11주차에서 큰 줄기로 언급한 컴플라이언스 매핑을 통제·증빙·SQL 수준으로 풉니다.

## 1. SOC 2 Type II — CC6.1, CC6.7, CC7.2

### CC6.1 (Logical Access)
"The entity implements logical access security software, infrastructure, and architectures over protected information assets..."

**우리 매핑:**
- 통제: 시크릿이 코드에 남지 않도록 4계층 방어선 + pre-receive hook (보충 3).
- 증빙: GitHub Actions run 기록, S3 보존 7년.
- 감사 SQL:

```sql
SELECT detector_name, COUNT(*) AS verified_findings
FROM trufflehog_audit_v1
WHERE verified = true
  AND scan_date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY detector_name;
```

### CC6.7 (System Operations - Anomalies)
- 통제: TruffleHog 알람을 SIEM에 흘려 anomaly 감지.
- 증빙: Splunk Notable Event 기록.

### CC7.2 (Incident Response)
- 통제: 10주차 플레이북.
- 증빙: 모든 인시던트의 포스트모템 모음.

## 2. PCI-DSS 4.0

### Req 8.3 (Strong Authentication)
- 매핑: 인증 자격 증명이 코드/리포에 평문으로 존재하지 않도록.
- 증빙: org-wide scan coverage 100% 보고.

### Req 8.6 (Programmatic Access)
- 매핑: API 키·서비스 계정 키 검출.
- 증빙: AWS, GCP 자격 증명 detector verified 통계.

### Req 10 (Logging & Monitoring)
- 매핑: 시크릿 스캐닝 결과를 변조 불가능한 형태로 1년 이상 보존.
- 증빙: S3 Object Lock retention 1y minimum, 우리는 7년.

### Req 6.3 (Software Development Security)
- 매핑: pre-commit + CI 통합.
- 증빙: `.github/workflows/trufflehog.yml` + pre-commit-config 코드.

## 3. HIPAA Security Rule §164.312

### (a)(1) Access Control
- 매핑: PHI 접근 자격 증명이 코드에 노출되지 않게.

### (b) Audit Controls
- 매핑: 시크릿 스캐닝 audit log를 변조 불가능하게.
- 증빙: WORM bucket + SHA256 ledger.

### (e)(1) Transmission Security
- 매핑: TLS 인증서·키 노출 검출 디텍터.

**HIPAA 보존**: 6년. 빅테크 표준 7년이 충분히 커버.

## 4. ISO 27001:2022 — Annex A.5/A.8

### A.5.17 (Authentication Information)
"Allocation and management of authentication information shall be controlled..."

### A.8.4 (Access to Source Code)
- 매핑: 소스코드 자체와 코드 안의 시크릿 모두 보호.

### A.8.10 (Information Deletion)
- 매핑: 시크릿이 누출된 경우 즉시 회수 + 히스토리 제거 (보충 1과 연결).

## 5. NIST 800-53 Rev 5 — IA-5, CM-7, AU-2

### IA-5 (Authenticator Management)
- 매핑: 시크릿 라이프사이클 관리. TruffleHog는 "Detection" 단계.

### CM-7 (Least Functionality)
- 매핑: verifier 토큰 등 도구 자체의 권한 최소화.

### AU-2 (Audit Events)
- 매핑: 시크릿 스캐닝 이벤트가 audit log에 포함.

## 6. 종합 매핑 표

| 우리 통제 | SOC 2 | PCI-DSS | HIPAA | ISO 27001 | NIST 800-53 |
|----------|-------|---------|-------|-----------|-------------|
| pre-commit hook | CC6.1 | 6.3 | §164.312(a)(1) | A.8.4 | IA-5 |
| CI block on verified | CC6.1 | 6.3 | §164.312(a)(1) | A.8.4 | IA-5 |
| Org-wide weekly scan | CC6.1, CC6.7 | 8.6 | §164.312(b) | A.5.17 | AU-2 |
| Splunk SIEM ingestion | CC6.7 | 10 | §164.312(b) | A.8.15 | AU-2 |
| Vault auto-rotate | CC6.1 | 8.3 | §164.312(a)(1) | A.5.17 | IA-5 |
| Incident playbook | CC7.2 | 12.10 | §164.308(a)(6) | A.5.26 | IR-4 |
| 7년 보존 (WORM) | CC6.7 | 10.7 | §164.316(b)(2) | A.5.34 | AU-11 |

## 7. 감사인이 자주 묻는 질문 (Top 10)

1. "시크릿 스캐닝 도구 결정은 어떻게 했나요?"
2. "Coverage가 100% 라는 증거는?"
3. "False Positive가 발생할 때 절차는?"
4. "alarm을 누가 받고 누가 처리하나요?"
5. "MTTR-revoke의 추세는?"
6. "도구의 자격 증명은 어떻게 관리되나요?"
7. "도구 자체가 침해되면 어떻게 알 수 있나요?"
8. "결과 무결성 보장 메커니즘은?"
9. "지난 1년간 verified 누출 사고 통계는?"
10. "정책 위반자에 대한 후속 조치는?"

각 질문마다 답변 + 증빙 위치를 미리 준비해 두는 것이 빅테크 표준.

## 8. 실습 과제

1. 회사 환경(가상)에 위 매핑 표를 그대로 채워 1쪽 PDF 작성.
2. Top 10 질문 모두에 1줄 답변 + 증빙 위치 작성.
3. SOC 2 CC6.1을 가정한 감사 SQL 3종 작성.
