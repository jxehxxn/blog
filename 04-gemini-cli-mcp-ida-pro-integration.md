# Gemini CLI와 IDA Pro 연동: 리버스 엔지니어링의 혁신

IDA Pro는 전 세계 리버스 엔지니어와 보안 연구원들이 사용하는 산업 표준 디스어셈블러 및 디버거입니다. Gemini CLI의 IDA-MCP를 사용하면, 복잡한 IDAPython 스크립트를 작성하는 대신 자연어로 IDA Pro의 강력한 분석 기능을 활용할 수 있습니다.

## 무엇을 할 수 있는가?

*   **바이너리 초기 분석 자동화**: "이 바이너리의 `main` 함수를 찾아서 그래프 뷰를 보여줘."
*   **코드 기능 설명**: "현재 함수의 의사코드(pseudocode)를 보여주고, 어떤 기능을 수행하는지 설명해줘."
*   **위험 함수 탐지**: "`strcpy`, `sprintf` 같이 잠재적으로 위험한 함수를 호출하는 모든 위치를 찾아줘."
*   **참조 검색**: "특정 문자열이나 함수가 어디에서 참조되는지 모두 알려줘."
*   **자동 주석 작성**: "이 함수가 어떤 인자를 받고 어떤 작업을 하는지 분석해서 주석을 추가해줘."

## 연동 준비

Playwright MCP와 달리, IDA-MCP는 IDA Pro 내부에서 플러그인 형태로 실행되어 Gemini CLI와 통신합니다.

1.  **IDA-MCP 스크립트 다운로드**: `gemini-cli` 공식 GitHub 레포지토리에서 `py/mcp/ida/gemini_ida_mcp.py` 파일을 다운로드합니다.
2.  **IDA 플러그인 디렉토리에 복사**: 다운로드한 `gemini_ida_mcp.py` 파일을 IDA Pro 설치 경로의 `plugins` 폴더에 복사합니다.
    *   예: `C:\Program Files\IDA Pro 7.7\plugins`
3.  **필요한 파이썬 라이브러리 설치**: IDA Pro에 내장된 파이썬 환경이나 시스템의 파이썬에 `websockets` 라이브러리를 설치해야 합니다.
    ```bash
    # IDA에 내장된 python.exe나 시스템 python.exe를 사용
    python.exe -m pip install websockets
    ```
4.  **IDA Pro에서 MCP 실행**:
    *   IDA Pro로 분석할 바이너리를 엽니다.
    *   `Edit -> Plugins -> Gemini IDA MCP` 메뉴를 선택하여 MCP 서버를 실행합니다.
    *   성공적으로 실행되면 IDA의 Output window에 "Gemini IDA MCP is running"과 같은 메시지가 표시됩니다.

## 실전 활용 예시

이제 Gemini CLI는 `ida`라는 도구를 사용할 수 있게 됩니다.

### 예시 1: 함수 분석 및 설명

```bash
gemini "ida를 사용해서 `sub_4010F0` 함수의 디컴파일된 코드를 보여주고, 이 함수가 어떤 역할을 하는지 단계별로 설명해줘."
```

Gemini는 IDA-MCP를 통해 IDA Pro의 디컴파일러(Hex-Rays)를 실행하고, 결과로 나온 의사코드를 가져와 분석한 뒤, 해당 함수의 기능(예: "이 함수는 사용자 입력을 받아 특정 키와 XOR 연산을 수행하는 암호화 루틴입니다.")을 설명해 줍니다.

### 예시 2: 상호 참조 찾기 (Xrefs)

```bash
gemini "ida, `CreateFileA` 함수를 호출하는 모든 곳의 주소를 알려줘."
```

IDA의 강력한 상호 참조(Xrefs) 기능을 자연어로 활용할 수 있습니다. Gemini는 해당 함수의 모든 호출 위치를 찾아 목록으로 정리해서 보여줍니다.

### 팁: IDA Pro 화면과 함께 보기

Gemini CLI의 터미널 출력과 함께 IDA Pro의 GUI 화면을 같이 보면 훨씬 효과적입니다. Gemini에게 특정 함수로 이동하라고 명령하면(`ida, 0x401234 주소로 점프해줘`), IDA의 그래프 뷰나 디스어셈블리 뷰가 해당 위치로 자동으로 이동하여 시각적인 분석을 돕습니다.

IDA-MCP는 리버스 엔지니어링의 진입 장벽을 낮추고, 숙련된 분석가에게는 반복적인 작업을 자동화하여 분석의 핵심에 더 집중할 수 있도록 돕는 혁신적인 도구입니다.
