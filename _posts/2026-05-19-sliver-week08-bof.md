---
layout: post
title: "Sliver Week 8: BOF (Beacon Object File)"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver senior
---

## 학습 목표

- BOF 개념과 Cobalt Strike에서 출범.
- Sliver의 BOF 지원.
- COFF loader 동작.
- 작성 + 실행.

## 1. BOF란

Cobalt Strike의 Beacon Object File. 작은 COFF (Windows obj) 파일을 beacon process 안에서 직접 실행. 새 process 생성 X → OpSec 강함.

Sliver도 COFF loader 통합.

## 2. 사용

```sliver
sliver (BEACON) > coff-loader load /path/to/whoami.o GO foo bar
```

또는 alias:
```sliver
sliver (BEACON) > whoami    # if installed
```

## 3. BOF Source

```c
// whoami.bof.c
#include <windows.h>
#include "beacon.h"

DECLSPEC_IMPORT BOOL WINAPI ADVAPI32$GetUserNameA(LPSTR, LPDWORD);

void go(char *args, int alen) {
    char name[256];
    DWORD len = 256;
    if (ADVAPI32$GetUserNameA(name, &len)) {
        BeaconPrintf(0x00, "Username: %s", name);
    }
}
```

## 4. Compile

```bash
x86_64-w64-mingw32-gcc -c whoami.bof.c -o whoami.x64.o
```

`.o` 파일을 server에 upload.

## 5. Why BOF?

- 새 process 생성 X (EDR 회피).
- 작음 (KB 단위).
- Windows API 직접 호출.
- Native execution.

## 6. BOF Library

CS BOF 호환:
- `BOF_Collection` (github).
- 사내 R&D 자체 BOF.

Sliver가 대부분 호환.

## 7. Limitation

- Windows only.
- 작아야 함.
- 디버깅 어려움.

## 8. Blue Team

- Process memory anomaly (코드 영역에 unmapped executable).
- API call sequence (suspicious).
- ETW (Event Tracing for Windows) ML.
- Defender ATP / CrowdStrike Falcon BOF detection.

## 9. 자가평가

### Q1. BOF 출범?
1. **Cobalt Strike**
2. Sliver 3. Metasploit 4. 무관

**정답: 1.**

### Q2. 새 process?
1. **생성 X (beacon process 안에서 실행)**
2. 매번 생성 3. UI 4. 무관

**정답: 1.**

### Q3. COFF?
1. **Windows obj 파일 형식**
2. C lib 3. UI 4. 무관

**정답: 1.**

### Q4. Compile?
1. **mingw cross-compile**
2. msbuild 3. UI 4. 무관

**정답: 1.**

### Q5. Blue team detection?
1. **process memory anomaly + ETW**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 9: Custom Extension].
