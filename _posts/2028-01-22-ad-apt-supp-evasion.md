---
layout: post
title: "Offensive AD & APT Mastery: Supplement - Advanced OpSec & Evasion (AMSI/ETW/Syscalls)"
---

Mimikatz를 그냥 실행하면? 1초 만에 백신이 잡아가고 보안팀에게 알람이 갑니다.
**OpSec (Operational Security)**과 **Evasion (탐지 우회)** 없이는 아무것도 할 수 없습니다.

최신 EDR(Endpoint Detection and Response)을 우회하기 위한 커널 레벨의 기술을 다룹니다.

---

## 1. AMSI Bypass (Anti-Malware Scan Interface)

윈도우는 PowerShell이나 스크립트가 실행될 때, 그 내용을 백신에게 보내서 검사받게 합니다. 이게 AMSI입니다.
"Invoke-Mimikatz"라는 문자열만 보여도 차단되죠.

**우회 원리:**
AMSI는 `amsi.dll`이라는 라이브러리를 통해 동작합니다.
이 DLL의 `AmsiScanBuffer` 함수를 패치해서 **"항상 깨끗함(Clean)"**이라고 리턴하게 만들면 됩니다.

```csharp
// Memory Patching (개념 코드)
var lib = LoadLibrary("amsi.dll");
var addr = GetProcAddress(lib, "AmsiScanBuffer");
// 해당 주소의 코드를 'return 0' (S_OK)으로 덮어씀.
```

---

## 2. ETW Patching (Event Tracing for Windows)

EDR은 윈도우의 감시 카메라망인 **ETW**를 통해 프로세스의 모든 행위를 봅니다.
C# 프로그램(`Assembly.Load`)을 실행하면 ETW 이벤트가 발생하고, EDR이 이를 잡습니다.

**우회 원리:**
AMSI와 똑같습니다. `ntdll.dll`의 `EtwEventWrite` 함수를 패치해서 로그가 전송되지 않게 만듭니다.
"카메라 렌즈에 테이프 붙이기"와 같습니다.

---

## 3. Direct Syscalls

EDR은 보통 `kernel32.dll`이나 `ntdll.dll` 같은 유저 모드 DLL에 **후킹(Hooking)**을 걸어둡니다.
우리가 `OpenProcess`를 호출하면, EDR의 감시 코드가 먼저 실행되고 나서 진짜 함수가 실행됩니다.

**우회 원리:**
EDR이 감시하고 있는 DLL을 거치지 않고, **직접 커널에게 명령(Syscall)**을 내립니다.
어셈블리어로 `syscall` 명령을 직접 호출하면 EDR은 우리가 뭘 했는지 알 수 없습니다.

*   **Tools:** SysWhispers2, SysWhispers3 (Syscall 스텁 생성기)

---

## 4. PPID Spoofing & Process Hollowing

*   **PPID Spoofing:** 악성코드를 실행할 때, 부모 프로세스를 `explorer.exe`나 `svchost.exe`인 것처럼 속입니다. (EDR을 혼란스럽게 함)
*   **Process Hollowing:** 정상적인 `notepad.exe`를 실행시킨 뒤, 메모리를 파내고(Hollow) 그 자리에 악성 코드를 채워 넣습니다. 작업 관리자에는 메모장으로 보입니다.

---

## 5. Summary

이 기술들은 창과 방패의 싸움입니다. EDR 벤더들도 계속 발전하고 있습니다.
하지만 원리(Hooking, Tracing)를 알면 우회할 길은 항상 있습니다.

**"Don't get caught."**
