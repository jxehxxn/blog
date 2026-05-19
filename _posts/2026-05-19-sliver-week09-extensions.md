---
layout: post
title: "Sliver Week 9: Custom Extensions (Go)"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver senior
---

## 학습 목표

- Sliver extension 모델.
- Aliases vs Extensions.
- Go로 작성.
- Armory (community catalog).

## 1. Aliases

기존 binary/script를 implant 안에서 실행. side-load.

```json
{
  "name": "sharphound",
  "command_name": "sharphound",
  "files": [
    {
      "os": "windows",
      "arch": "amd64",
      "path": "SharpHound.exe"
    }
  ]
}
```

`alias install sharphound.alias`.

## 2. Extensions (Sliver native)

Go shared lib (.so / .dll). implant 안에서 load.

```go
package main

import (
    "C"
    "os/user"
)

//export Whoami
func Whoami() *C.char {
    u, _ := user.Current()
    return C.CString(u.Username)
}

func main() {}
```

```bash
go build -buildmode=c-shared -o whoami.so whoami.go
```

## 3. Extension Manifest

```json
{
  "name": "my-ext",
  "command_name": "my-ext",
  "version": "1.0.0",
  "extension_author": "alice",
  "files": [
    { "os": "windows", "arch": "amd64", "path": "ext.dll" }
  ],
  "init": "InitExt",
  "entrypoint": "Whoami"
}
```

`extensions install my-ext.json`.

## 4. Armory

Sliver community catalog. `armory install <package>` 같은 식.

표준 alias/extension 다수.

## 5. 사내 R&D 패턴

- 사내 자체 BOF/extension 작성.
- 보안팀 review.
- private armory.

## 6. Limitation

- Extension은 implant 안에서 실행 → process 영향.
- 큰 extension은 메모리 증가.
- Debug 어려움.

## 7. Blue Team

- 사이드로드된 binary (suspicious DLL load).
- Unusual API call sequence.
- Go-compiled extension의 strings.

## 8. 자가평가

### Q1. Alias vs Extension?
1. **Alias 기존 binary, Extension native lib**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q2. Armory?
1. **Sliver community catalog**
2. UI 3. 무관 4. 빠름

**정답: 1.**

### Q3. Extension build?
1. **go build c-shared**
2. UI 3. msbuild 4. 무관

**정답: 1.**

### Q4. Manifest entrypoint?
1. **호출 함수 export 이름**
2. UI 3. 무관 4. main

**정답: 1.**

### Q5. Blue team?
1. **suspicious DLL load + API anomaly**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 10: Blue Team IoC + Detection].
