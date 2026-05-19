---
layout: post
title: "Sliver 보충 3: EDR Evasion — 방어자가 알아야 할 패턴"
date: 2026-05-19 11:00:00 +0900
categories: security c2 sliver edr supplement
---

본 글은 **방어자가 공격자의 회피 기법을 알아야 더 잘 막을 수 있다**는 전제로 작성. 비인가 사용 금지.

## EDR이 보는 것

1. Process create (parent-child chain).
2. Image load (DLL).
3. Network connect.
4. File create/modify.
5. Registry change.
6. ETW events (Windows).
7. API call sequence (in-line hook).
8. Memory anomaly (unmapped executable).

## 공격자 회피 기법 (탐지 관점)

### Process Hollowing
정상 process 띄움 → 메모리 교체. 새 process 생성 자체는 정상이라 탐지 어려움.
**방어**: ETW + memory scan + image integrity.

### Reflective DLL Injection
disk 안 거치고 메모리 안에서 DLL load.
**방어**: ETW + AMSI + memory hunting.

### Direct Syscall
ntdll의 syscall 직접 호출 → user-mode hook 우회.
**방어**: kernel-mode ETW (ETW-TI).

### AMSI Bypass
PowerShell / .NET malware scanning 우회.
**방어**: AMSI signature update + behavioral.

### Sleep Encryption (Foliage, Ekko)
beacon sleep 중 메모리 암호화 → memory scan 회피.
**방어**: 적극적 메모리 sampling.

### Indirect Syscall (Hell's Gate, Halo's Gate)
Direct syscall + 동적 SSN (System Service Number) 추출.
**방어**: kernel ETW.

## 방어 도구

- CrowdStrike Falcon.
- SentinelOne.
- MS Defender ATP (E5).
- Carbon Black.
- Elastic Security.

## Detection Engineering

각 회피 기법별 detection rule 작성. **detection gap**을 알면 우선순위 매김.

## 결론

방어자는 회피 기법을 학습해야 detection 우선순위 + EDR 평가 가능. 윤리적 사용 한정.
