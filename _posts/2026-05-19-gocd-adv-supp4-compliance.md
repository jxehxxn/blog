---
layout: post
title: "GoCD 심화 보충 4: SOC 2 / PCI / SOX 매핑"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd compliance supplement
---

## SOC 2 CC8.1 (Change Management)
- PR + reviewer (branch protection).
- Manual approval audit (GoCD log).
- Pipeline as Code (Git history).

## PCI-DSS 6.3
- Secure development.
- Manual approval for cardholder env.
- Secret plugin (no plaintext).

## SOX (US public company)
- Audit trail.
- 7년 보존.
- Segregation of duty (deploy approver ≠ developer).

## Evidence 자동
- GoCD audit log → SIEM (Splunk).
- Manual approval 기록 → S3 WORM 7년.
- Pipeline change history.

## 감사 질문 Top 5
1. "Production deploy approval 누가?"
2. "Secret 어디 저장?"
3. "Pipeline 변경 추적?"
4. "Backup + DR drill?"
5. "Access 분리?"

각 답변 + evidence URL.

## 결론
GoCD의 manual approval first-class가 audit에 매우 유리.
