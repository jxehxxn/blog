# Gemini CLI와 Playwright 연동하기: 웹 자동화의 새로운 차원

Playwright는 Microsoft에서 개발한 강력한 웹 브라우저 자동화 라이브러리입니다. Gemini CLI의 Playwright MCP와 연동하면, 복잡한 코드를 작성할 필요 없이 자연어로 웹 브라우저를 제어하여 다양한 작업을 수행할 수 있습니다.

## 무엇을 할 수 있는가?

*   **웹 스크래핑 및 데이터 추출**: "이 뉴스 사이트에서 최신 기사 10개의 제목과 링크를 가져와줘."
*   **E2E(End-to-End) 테스트**: "로그인 페이지로 가서, 이 아이디와 비밀번호로 로그인한 뒤, '내 정보' 페이지가 제대로 표시되는지 확인해줘."
*   **반복적인 웹 작업 자동화**: "매일 아침 이 시스템에 로그인해서 리포트를 다운로드해줘."
*   **웹사이트 상호작용**: "이 설문조사 페이지에서 첫 번째 질문에는 'A', 두 번째 질문에는 'C'를 선택하고 제출해줘."

## 연동 준비

1.  **Node.js 및 npm 설치**: Playwright는 Node.js 기반이므로, 시스템에 Node.js와 npm이 설치되어 있어야 합니다.
2.  **Playwright 설치**: Gemini CLI 프로젝트와는 별개로, Playwright를 사용할 프로젝트 폴더를 만들고 그 안에서 Playwright를 설치합니다.

    ```bash
    mkdir my-playwright-project
    cd my-playwright-project
    npm init -y
    npm i -D playwright
    npx playwright install # Chromium, Firefox, WebKit 등 브라우저 설치
    ```

3.  **Playwright MCP 실행**: Gemini CLI가 제어할 수 있도록 Playwright MCP 서버를 실행해야 합니다. `gemini-cli` 레포지토리의 `js/mcp/playwright` 디렉토리에서 서버를 실행합니다.

    ```bash
    # gemini-cli 레포지토리 클론 후 해당 디렉토리로 이동
    cd path/to/gemini-cli/js/mcp/playwright
    npm install
    npm start
    ```

    서버가 실행되면 Gemini CLI가 해당 MCP를 감지하고 `playwright`라는 도구로 사용할 수 있게 됩니다.

## 실전 활용 예시

이제 Gemini CLI 프롬프트에서 `playwright` 도구를 사용하여 웹 브라우저를 제어할 수 있습니다.

### 예시 1: 특정 사이트 정보 검색

```bash
gemini "playwright를 사용해서 duckduckgo.com에 접속한 다음, 'Gemini CLI'를 검색하고 결과 페이지의 스크린샷을 찍어줘."
```

Gemini는 이 명령을 해석하여 다음과 같은 일련의 작업을 Playwright MCP를 통해 수행합니다.
1.  새 브라우저 페이지를 엽니다.
2.  `https://duckduckgo.com`으로 이동합니다.
3.  검색창을 찾아 'Gemini CLI'를 입력합니다.
4.  검색 버튼을 클릭합니다.
5.  결과 페이지가 로드되면 스크린샷을 찍어 로컬 파일로 저장하고, 그 경로를 알려줍니다.

### 예시 2: 데이터 추출

```bash
gemini "playwright를 써서 해커뉴스(news.ycombinator.com)에 방문해서, 현재 페이지에 있는 상위 5개 포스트의 제목(title)과 URL을 알려줘."
```

Gemini는 페이지의 HTML 구조를 분석하여 `<a>` 태그와 특정 CSS 클래스를 가진 요소들을 찾아내고, 그 안의 텍스트와 `href` 속성을 추출하여 결과를 정리해서 보여줍니다.

### 팁: `use` 명령어로 컨텍스트 고정

매번 `playwright`를 입력하는 것이 번거롭다면 `use` 명령어로 기본 도구를 설정할 수 있습니다.

```bash
# playwright 도구를 기본으로 사용하도록 설정
use playwright

# 이제 'playwright'를 명시하지 않아도 됨
gemini "google.com으로 가서 '오늘 날씨'를 검색해줘"
```

Playwright MCP는 Gemini CLI의 활용 범위를 데스크톱에서 웹으로 확장시키는 강력한 기능입니다. 단순 반복적인 웹 작업을 자동화하고 싶거나, 웹 기반의 정보를 수집하고 싶을 때 적극적으로 활용해 보세요.
