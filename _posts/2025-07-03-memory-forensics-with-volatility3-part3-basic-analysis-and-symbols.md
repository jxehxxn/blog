---
layout: post
title:  "메모리 덤프 파헤치기 (3/5) - 기본 분석과 심볼"
date:   2025-07-03 10:00:00 +0900
categories: forensics volatility
---

## 들어가며

지난 2부에서 Volatility3 분석 환경을 성공적으로 구축했습니다. 이제 준비된 `memdump.mem` 파일을 가지고 본격적인 분석의 세계로 뛰어들 차례입니다. 이번 3부에서는 덤프 파일의 기본적인 정보를 확인하는 방법과, 메모리 분석의 핵심이자 Volatility3의 가장 큰 장점 중 하나인 **심볼(Symbol)**에 대해 깊이 있게 알아보겠습니다.

## 1. 첫 분석: 덤프 정보 확인하기

분석의 첫 단계는 우리가 가진 메모리 덤프가 어떤 운영체제(OS)에서, 언제, 어떻게 생성되었는지 파악하는 것입니다. Volatility3에서는 `windows.info.Info` 플러그인을 사용해 이 정보를 손쉽게 확인할 수 있습니다.

-   **실행 명령어:** `vol.py -f <메모리덤프_파일경로> <플러그인_이름>`
-   `-f` 옵션: 분석할 메모리 덤프 파일을 지정합니다.

```bash
# C:\volatility_workspace 에서 실행
# vol.py의 전체 경로를 지정하거나, PATH에 등록했다면 vol.py만 사용
python .\venv\Lib\site-packages\volatility3\vol.py -f memdump.mem windows.info.Info
```

**실행 결과 예시:**

```
Volatility 3 Framework 2.x.x
Progress:  100.00		PDB scanning finished
Variable	Value

Kernel Base	0xf8011a204000
DTB	0x1ad000
Symbols	file:///<...>/volatility3/symbols/windows/win10x64_19041.json.xz
Is 64-bit	True
Is PAE	True
Number of Processors	4
System Time	2025-07-10 01:15:00 UTC+0000
System Uptime	0 days, 01:23:45.678
...
```

위 결과에서 우리는 다음과 같은 중요한 정보를 얻을 수 있습니다.

-   **Symbols:** Volatility3가 이 메모리 덤프를 분석하기 위해 `win10x64_19041` 버전에 맞는 심볼 파일을 사용했음을 알려줍니다.
-   **Is 64-bit:** 64비트 운영체제임을 알 수 있습니다.
-   **System Time:** 메모리 덤프가 생성될 당시의 시스템 시간(UTC 기준)을 보여줍니다.

Volatility 2에서는 `imageinfo` 플러그인으로 OS 프로파일을 추측하고, 사용자가 직접 프로파일을 지정해야 하는 번거로움이 있었습니다. 하지만 **Volatility 3는 이 과정을 완전히 자동화**하여 `windows.info.Info` 또는 다른 Windows 플러그인을 실행하면 스스로 적절한 심볼을 찾아 분석을 진행합니다.

## 2. 심볼(Symbol)이란 무엇인가?

심볼은 메모리 분석의 핵심 개념입니다. 간단히 말해, **심볼은 운영체제 커널의 데이터 구조에 대한 '지도'**입니다.

-   **데이터 구조(Data Structure):** 운영체제는 프로세스 정보(EPROCESS), 쓰레드 정보(ETHREAD), 파일 정보(FILE_OBJECT) 등을 특정한 '틀'에 맞춰 관리합니다. 이 틀을 데이터 구조라고 합니다.
-   **심볼 테이블(Symbol Table):** 어떤 데이터가 메모리상의 어느 주소에 어떤 구조로 저장되어 있는지를 정의한 표입니다. 예를 들어, "프로세스 목록의 시작점은 `PsActiveProcessHead`라는 이름으로 0x... 주소에 저장되어 있다" 와 같은 정보가 담겨있습니다.

Volatility는 이 심볼 '지도'를 보고 메모리 덤프라는 거대한 데이터의 바다에서 우리가 원하는 정보(프로세스 목록, 네트워크 연결 등)를 정확하게 찾아낼 수 있습니다.

## 3. Volatility3의 똑똑한 심볼 관리

Volatility3의 가장 큰 혁신은 바로 심볼 관리 방식에 있습니다.

### 자동 다운로드

Volatility3는 분석 시점에 해당 OS 버전에 맞는 심볼 파일이 로컬에 없으면, **자동으로 Microsoft나 Mozilla의 심볼 서버에 접속하여 필요한 심볼을 다운로드**합니다. 다운로드된 심볼은 JSON 형태로 변환되어 다음 경로에 저장됩니다.

-   `volatility3/symbols/windows/`
-   `volatility3/symbols/linux/`
-   `volatility3/symbols/mac/`

이 기능 덕분에 사용자는 더 이상 OS 버전마다 맞는 프로파일을 찾아 헤맬 필요가 없어졌습니다.

### 오프라인 환경 또는 심볼 수동 지정

만약 분석 환경이 인터넷에 연결되지 않은 폐쇄망이거나, 자동 탐지에 실패하는 경우 수동으로 심볼을 제공할 수 있습니다.

1.  **심볼 팩 다운로드:** 인터넷이 되는 다른 PC에서 Volatility Team이 미리 만들어둔 심볼 팩을 다운로드합니다. (웹에서 `Volatility 3 Symbol Packs` 검색)
2.  **심볼 폴더에 복사:** 다운로드한 심볼 파일(`windows.zip`, `linux.zip` 등)의 압축을 풀어 위에서 언급한 `volatility3/symbols/` 폴더에 넣어줍니다.
3.  **수동 지정 (필요시):** 대부분의 경우 심볼 폴더에 넣어두기만 하면 자동으로 인식하지만, 특정 심볼을 강제로 사용하고 싶다면 플러그인 이름 뒤에 `--symbols` 옵션을 추가할 수 있습니다.

    ```bash
    vol.py -f memdump.mem --symbols file:///path/to/your/symbol.json.xz windows.pslist.PsList
    ```

---

이제 우리는 메모리 덤프의 기본 정보를 확인하고, 분석의 원리를 떠받치는 '심볼'의 개념까지 이해했습니다. Volatility3가 어떻게 이 심볼을 스마트하게 관리하는지 알게 된 것이 이번 파트의 가장 큰 수확입니다.

다음 4부에서는 드디어 본격적으로 메모리 속 정보를 캐내는 작업을 시작하겠습니다. 프로세스, 네트워크, 레지스트리 등 시스템의 핵심 정보를 추출하는 다양한 플러그인들의 사용법을 자세히 다룰 예정이니 많은 기대 바랍니다.
