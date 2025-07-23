# ida-pro-mcp 시작하기: AI와 함께하는 리버스 엔지니어링

`mrexodia/ida-pro-mcp`는 IDA Pro를 Gemini와 같은 대규모 언어 모델(LLM)과 연결하여, 지루하고 반복적인 리버스 엔지니어링 작업을 자동화하는 강력한 MCP(Managed Code Piper) 서버입니다. 이 도구를 사용하면 자연어 명령만으로 IDA Pro의 깊이 있는 기능들을 활용할 수 있습니다.

## 주요 기능

*   **자연어 기반 분석**: "이 함수의 기능을 설명해줘" 와 같이 간단한 명령으로 복잡한 분석 작업을 수행합니다.
*   **정적 분석 자동화**: 함수 디컴파일, 변수 이름 변경, 주석 추가, 구조체 생성 등 다양한 정적 분석 작업을 자동화합니다.
*   **동적 분석 (디버깅) 지원**: 디버거와 연동하여 브레이크포인트 설정, 레지스터 및 메모리 덤프, 실행 제어 등을 수행할 수 있습니다.
*   **유연한 확장성**: IDA Pro의 거의 모든 IDAPython API에 접근할 수 있어, 사용자가 원하는 기능을 쉽게 추가하고 확장할 수 있습니다.

## 설치 및 설정

`ida-pro-mcp`는 설치 과정을 자동화하는 스크립트를 제공하여 매우 편리합니다.

1.  **PowerShell에서 설치 스크립트 실행**:
    IDA Pro가 설치된 환경의 PowerShell에서 아래 명령어를 실행하면 리포지토리 클론부터 필요한 라이브러리 설치, IDA 플러그인 폴더로의 복사까지 모든 과정이 자동으로 처리됩니다.

    ```powershell
    iex (irm https://raw.githubusercontent.com/mrexodia/ida-pro-mcp/main/install.ps1)
    ```

2.  **IDA Pro에서 플러그인 실행**:
    *   IDA Pro로 분석할 파일을 엽니다.
    *   상단 메뉴에서 `Edit -> Plugins -> mcp-ida-pro`를 선택하여 MCP 서버를 실행합니다.
    *   IDA의 Output window에 "[mcp-ida-pro] Listening on port 8000..." 과 같은 메시지가 나타나면 성공적으로 실행된 것입니다.

## Gemini CLI와 연동

이제 Gemini CLI가 `ida`라는 도구를 사용하여 이 MCP 서버와 통신할 수 있습니다. 별도의 설정 없이 MCP 서버가 실행되면 Gemini CLI가 자동으로 감지합니다.

### 연동 확인

Gemini CLI 프롬프트에서 간단한 명령으로 연동을 테스트할 수 있습니다.

```bash
gemini "ida를 사용해서 현재 열려있는 파일의 이름과 MD5 해시를 알려줘."
```

Gemini가 IDA Pro로부터 파일 정보를 가져와 정확한 답변을 한다면, 성공적으로 연동된 것입니다.

이제 AI와 함께 리버스 엔지니어링을 수행할 준비가 모두 끝났습니다. 다음 포스트에서는 `ida-pro-mcp`를 활용한 정적 분석 자동화 기법에 대해 자세히 알아보겠습니다.
