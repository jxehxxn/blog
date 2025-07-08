---
layout: post
title:  "핵심 플러그인 활용 (4/5) - 프로세스, 네트워크, 레지스트리"
date:   2025-07-04 10:00:00 +0900
categories: forensics volatility
---

## 들어가며

지난 3부까지 우리는 Volatility3 환경을 구축하고, 메모리 덤프의 기본 정보와 핵심 개념인 '심볼'에 대해 알아보았습니다. 이제부터가 진짜입니다. 이번 4부에서는 메모리 포렌식의 '꽃'이라 할 수 있는 핵심 플러그인들을 사용하여, 시스템의 살아있는 정보를 직접 추출하고 분석하는 방법을 다룹니다.

가장 기본적이면서도 중요한 **프로세스, 네트워크, 레지스트리** 관련 플러그인들을 집중적으로 실습해보겠습니다. 모든 명령어는 `C:\volatility_workspace`에서 `memdump.mem` 파일을 대상으로 실행한다고 가정합니다.

## 1. 프로세스 분석 플러그인

시스템에서 어떤 프로그램이 실행 중이었는지 확인하는 것은 분석의 가장 첫 단계입니다.

### `windows.pslist.PsList` - 프로세스 목록 확인

가장 기본적인 플러그인으로, 메모리 덤프 시점에 실행 중이던 프로세스의 목록을 보여줍니다.

```bash
vol.py -f memdump.mem windows.pslist.PsList
```

**주요 출력 항목:**
- **PID:** 프로세스 ID
- **PPID:** 부모 프로세스 ID
- **ImageFileName:** 프로세스 실행 파일 이름
- **Offset(V):** 프로세스 객체(EPROCESS)의 가상 주소
- **Threads:** 쓰레드 개수
- **CreateTime:** 프로세스 생성 시간

> **분석 Tip:** `svchost.exe`와 같이 정상적인 시스템 프로세스처럼 보이지만, PPID가 비정상적이거나(예: `explorer.exe`가 아닌 다른 프로세스) 이름에 오타가 있는 경우(예: `svch0st.exe`) 악성 프로세스를 의심해볼 수 있습니다.

### `windows.pstree.PsTree` - 프로세스 관계 확인

`pslist`와 비슷하지만, 프로세스 간의 부모-자식 관계를 트리 구조로 보여주어 관계를 파악하기 훨씬 용이합니다.

```bash
vol.py -f memdump.mem windows.pstree.PsTree
```

**분석 Tip:** 어떤 프로세스가 다른 프로세스를 실행시켰는지 한눈에 파악할 수 있습니다. 예를 들어, `powershell.exe`이 알 수 없는 이름의 프로세스를 실행시켰다면 의심스러운 행위일 수 있습니다.

### `windows.cmdline.CmdLine` - 프로세스 실행 인수 확인

프로세스가 어떤 옵션(인수)으로 실행되었는지 보여줍니다. 악성코드는 종종 특정 옵션을 사용하여 자신을 실행시키므로 매우 중요한 정보입니다.

```bash
vol.py -f memdump.mem windows.cmdline.CmdLine
```

**분석 Tip:** `svchost.exe -k netsvcs` 와 같은 정상적인 형태가 아닌, 이상한 스크립트나 암호화된 문자열을 인자로 실행하는 프로세스를 찾아보세요.

### `windows.psscan.PsScan` - 숨겨진 프로세스 탐지

`pslist`가 정상적인 방법으로 프로세스 목록을 가져오는 반면, `psscan`은 메모리 전체를 스캔하여 프로세스 구조체(EPROCESS)의 특징을 가진 데이터를 모두 찾아냅니다. 이를 통해 루트킷(Rootkit) 등에 의해 은닉된 프로세스를 탐지할 수 있습니다.

```bash
vol.py -f memdump.mem windows.psscan.PsScan
```

**분석 Tip:** `pslist` 결과에는 없는데 `psscan` 결과에는 나타나는 프로세스가 있다면, 해당 프로세스는 악성코드에 의해 숨겨졌을 확률이 매우 높습니다.

## 2. 네트워크 분석 플러그인

### `windows.netscan.NetScan` - 네트워크 연결 정보 확인

메모리 덤프 시점의 활성화된 네트워크 연결(TCP/UDP) 정보를 보여줍니다. 어떤 프로세스가 어느 IP 주소와 통신했는지 알 수 있습니다.

```bash
vol.py -f memdump.mem windows.netscan.NetScan
```

**주요 출력 항목:**
- **Offset(V):** 네트워크 객체의 가상 주소
- **Proto:** 프로토콜 (TCP/UDP)
- **LocalAddr:** 로컬 주소
- **LocalPort:** 로컬 포트
- **ForeignAddr:** 외부 주소
- **ForeignPort:** 외부 포트
- **State:** 연결 상태 (LISTENING, ESTABLISHED, TIME_WAIT 등)
- **PID:** 연결을 맺은 프로세스의 ID
- **Owner:** 프로세스 이름

**분석 Tip:** 알려지지 않은 외부 IP나 비정상적인 포트와 통신하는 프로세스가 있다면 악성코드의 C&C(Command & Control) 서버 통신일 수 있습니다.

## 3. 레지스트리 분석 플러그인

레지스트리는 Windows 시스템의 모든 설정 정보가 담긴 데이터베이스입니다. 악성코드는 지속성 유지(부팅 시 자동 실행)를 위해 레지스트리를 조작하는 경우가 많습니다.

### `windows.registry.hivelist.HiveList` - 레지스트리 하이브 목록 확인

메모리에 로드된 레지스트리 하이브(Hive) 파일의 목록과 가상 주소, 실제 파일 경로를 보여줍니다.

```bash
vol.py -f memdump.mem windows.registry.hivelist.HiveList
```

**분석 Tip:** `printkey` 플러그인을 사용하려면 여기서 확인한 하이브의 가상 주소(Offset)가 필요합니다.

### `windows.registry.printkey.PrintKey` - 레지스트리 키/값 확인

특정 레지스트리 키의 하위 키와 값들을 출력합니다.

```bash
# 먼저 hivelist로 SAM 하이브의 주소를 찾았다고 가정 (예: 0xdeadbeef)
vol.py -f memdump.mem --offset 0xdeadbeef windows.registry.printkey.PrintKey --key "SAM\Domains\Account\Users"
```

**분석 Tip:** 악성코드가 자동 실행을 위해 등록하는 다음 경로들을 확인해보는 것이 좋습니다.
- `SOFTWARE\Microsoft\Windows\CurrentVersion\Run`
- `SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce`

---

이번 4부에서는 Volatility3의 핵심 플러그인들을 사용하여 실제 분석과 유사한 경험을 해보았습니다. 이 플러그인들의 결과를 서로 연관지어 분석하면(예: `netscan`에서 의심스러운 프로세스 PID를 발견 -> `pslist`에서 해당 PID 정보 확인 -> `cmdline`으로 실행 인수 확인) 훨씬 더 강력한 분석이 가능합니다.

이제 마지막 5부에서는 가상의 악성코드 감염 시나리오를 설정하고, 지금까지 배운 모든 기술을 총동원하여 침해사고 분석을 처음부터 끝까지 수행하는 실전 훈련을 진행하겠습니다.
