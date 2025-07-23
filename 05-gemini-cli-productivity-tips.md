# Gemini CLI 생산성 팁: 나만의 명령어 추가 및 활용법

Gemini CLI를 사용하다 보면 반복적으로 입력하는 길고 복잡한 명령어가 생기기 마련입니다. `settings.json` 파일에 자신만의 명령어를 추가(Alias)하여 생산성을 크게 향상시킬 수 있습니다.

## 1. 나만의 명령어(Alias) 만들기

Gemini CLI의 설정 파일(`~/.gemini/settings.json` 또는 `C:\Users\사용자명\.gemini\settings.json`)에 `aliases` 항목을 추가하여 단축키처럼 사용할 수 있는 명령어를 등록할 수 있습니다.

### 설정 파일 열기

먼저 Gemini CLI의 설정 디렉토리로 이동하여 `settings.json` 파일을 엽니다.

### Alias 추가하기

`settings.json` 파일에 다음과 같은 형식으로 `aliases` 객체를 추가하거나 수정합니다.

```json
{
  "aliases": {
    "explain": "read_file \"$1\" | gemini \"이 코드의 기능을 자세히 설명해줘. 특히, 각 함수의 역할과 상호작용에 초점을 맞춰줘.\"",
    "refactor": "read_file \"$1\" | gemini \"이 코드를 더 읽기 쉽고 효율적으로 리팩토링해줘. 변경된 부분에 대한 설명도 덧붙여줘.\"",
    "commit": "git diff --cached | gemini \"이 변경사항에 대한 git 커밋 메시지를 Conventional Commits 양식에 맞춰서 작성해줘. 타입은 feat, fix, docs, style, refactor, test, chore 중에서 선택해줘.\"",
    "translate_ko": "gemini \"다음 영문을 한국어로 번역해줘: $1\""
  }
}
```

*   `"explain"`: `gemini explain <파일경로>` 형식으로 사용하며, 지정된 파일을 읽어 Gemini에게 설명을 요청합니다. `$1`은 명령어 뒤에 오는 첫 번째 인자를 의미합니다.
*   `"refactor"`: `gemini refactor <파일경로>` 형식으로 사용하며, 코드 리팩토링을 요청합니다.
*   `"commit"`: `gemini commit` 이라고만 입력하면, `git add`로 스테이징된 변경사항을 바탕으로 Gemini가 커밋 메시지를 추천해줍니다.
*   `"translate_ko"`: `gemini translate_ko "Hello, world!"` 와 같이 간단한 번역기로 활용할 수 있습니다.

### Alias 사용하기

이제 터미널에서 새로 등록한 명령어를 바로 사용할 수 있습니다.

```bash
# explain 별칭을 사용하여 파일 설명 요청
gemini explain "C:\Users\note\Desktop\dev\my-project\src\main.py"

# commit 별칭을 사용하여 커밋 메시지 생성
# (git add . 로 변경사항을 스테이징한 후 실행)
gemini commit
```

## 2. 프롬프트 엔지니어링 팁

Gemini CLI의 성능은 프롬프트의 품질에 크게 좌우됩니다. 더 좋은 결과를 얻기 위한 몇 가지 팁입니다.

*   **역할 부여 (Persona)**: 프롬프트 시작 부분에 Gemini의 역할을 지정해주면 더 전문적인 답변을 얻을 수 있습니다.
    > "너는 20년 경력의 C++ 개발자야. 아래 코드를 보고..."
*   **컨텍스트 제공**: `read_file`이나 `read_many_files`를 적극적으로 활용하여 질문에 필요한 충분한 컨텍스트를 제공하세요. 파일 하나만 덜렁 주는 것보다, 관련 헤더 파일이나 호출하는 함수의 코드를 같이 제공하면 훨씬 정확한 답변이 나옵니다.
*   **단계별 요청**: 한 번에 너무 많은 것을 요구하지 마세요. "코드를 분석하고, 리팩토링하고, 테스트 케이스까지 작성해줘" 보다는 "코드 분석 -> 리팩토링 -> 테스트 케이스 작성" 순으로 작업을 나누어 요청하는 것이 좋습니다.
*   **결과 형식 지정**: 원하는 결과물의 형식을 구체적으로 명시하면 좋습니다.
    > "... 결과를 JSON 형식으로 알려줘.", "... 답변을 마크다운 테이블로 정리해줘."

## 3. `use` 명령어로 컨텍스트 유지

MCP를 사용하거나 특정 파일에 대해 연속적인 질문을 할 때, `use` 명령어로 기본 도구나 파일을 고정하면 편리합니다.

```bash
# playwright MCP를 기본 도구로 설정
use playwright
gemini "google.com으로 이동"
gemini "검색창에 'Gemini CLI' 입력"

# 특정 파일을 기본 컨텍스트로 설정
use file "C:\Users\note\Desktop\dev\my-project\src\main.py"
gemini "이 파일의 50번째 줄에 있는 함수 이름이 뭐야?"
gemini "그 함수의 역할을 설명해줘"
```

이러한 팁들을 활용하여 Gemini CLI를 자신만의 강력한 개발 보조 도구로 만들어보세요.
