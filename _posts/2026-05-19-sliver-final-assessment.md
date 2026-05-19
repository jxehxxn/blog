---
layout: post
title: "Sliver 최종 자가평가 — 30문항 종합"
date: 2026-05-19 11:00:00 +0900
categories: security c2 sliver assessment senior
---

각 정답 1.

## 기초 / 윤리
### Q1. C2? **Command and Control**.
### Q2. Sliver 모기업? **BishopFox**.
### Q3. Sliver vs CS? **Sliver OSS, CS 상용**.
### Q4. 합법적 사용 핵심? **authorized engagement + 본인 소유 + CTF**.
### Q5. Purple team? **Red + Blue 협업**.

## 운영
### Q6. Server data dir? **~/.sliver/**.
### Q7. mTLS 강점? **양방향 cert 인증**.
### Q8. DNS C2 단점? **느림 (KB/s)**.
### Q9. WireGuard? **VPN-like channel**.
### Q10. Multiple listener? **primary 실패 시 fallback**.
### Q11. Format 4종? **exe/shared/service/shellcode**.
### Q12. Profile? **옵션 표준화**.
### Q13. Stager? **작은 → 큰 implant**.
### Q14. Session vs Beacon? **상시 vs 주기**.
### Q15. Jitter? **callback 간격 변동**.
### Q16. Persistence 3종? **registry/service/scheduled task**.
### Q17. portfwd vs socks5? **단일 포트 vs 모든 traffic**.
### Q18. SMB Pivot? **named pipe**.
### Q19. BOF 출범? **Cobalt Strike**.
### Q20. BOF 강점? **새 process X**.

## 방어
### Q21. JA3? **TLS client 핑거프린트**.
### Q22. Sysmon event 3? **network connect**.
### Q23. Falco? **Linux runtime**.
### Q24. ETW-TI? **kernel-mode ETW (direct syscall 탐지)**.
### Q25. Sleep encryption? **memory scan 회피**.
### Q26. AMSI? **PowerShell/.NET scanning**.
### Q27. ATT&CK T1071.004? **DNS**.
### Q28. ATT&CK T1055? **Process Injection**.
### Q29. Detection KPI? **MTTD + FP rate**.
### Q30. ROE? **Rules of Engagement (서면 동의)**.

## 채점
- 28~30: Senior Offensive / Detection 수준.
- 24~27: 미드.
- 18~23: 주니어.
- ~17: 핵심 재복습.

법·윤리 재확인: 본 강의로 학습한 기술은 authorized 환경에서만 사용. 위반 시 형사 책임.
