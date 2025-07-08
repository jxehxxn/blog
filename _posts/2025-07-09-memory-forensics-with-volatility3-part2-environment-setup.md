---
layout: post
title:  "완벽한 분석 환경 구축하기 (2/5)"
date:   2025-07-09 10:00:00 +0900
categories: forensics volatility
---

## 들어가며

지난 1부에서는 메모리 포렌식의 개념과 Volatility3에 대해 간단히 알아보았습니다. 오늘은 본격적인 실습에 앞서 가장 중요한 단계인 **분석 환경 구축**을 진행하겠습니다. 어떤 일이든 연장이 좋아야 작업이 수월한 법이죠. Volatility3를 설치하고 실습용 메모리 덤프 파일을 준비하는 과정을 차근차근 따라 해보세요.

## 1. 사전 준비물: Python 3

Volatility3는 Python 3로 만들어졌습니다. 따라서 여러분의 컴퓨터에 Python 3.6 이상의 버전이 설치되어 있어야 합니다.

### Windows에서 Python 설치하기

1.  **Python 공식 홈페이지 접속:** [python.org](https://python.org)에 접속하여 `Downloads` 메뉴로 이동합니다.
2.  **최신 버전 다운로드:** 'Download Python 3.x.x' 버튼을 클릭하여 설치 파일을 다운로드합니다.
3.  **설치 진행:**
    - 설치 파일을 실행합니다.
    - **매우 중요:** 설치 첫 화면에서 **`Add Python 3.x to PATH`** 체크박스를 **반드시** 선택해주세요. 이것을 체크해야 명령 프롬프트(cmd)나 PowerShell 어디서든 `python` 명령어를 사용할 수 있습니다.
    - `Install Now`를 클릭하여 설치를 진행합니다.
4.  **설치 확인:** 설치가 완료되면, 명령 프롬프트를 열고 아래 명령어를 입력하여 설치된 버전을 확인합니다.

```bash
python --version
pip --version
```

위 명령어를 입력했을 때 Python과 pip의 버전 정보가 잘 보인다면 성공적으로 설치된 것입니다.

## 2. Volatility3 설치하기

Python이 준비되었다면 Volatility3 설치는 매우 간단합니다. `pip`이라는 Python 패키지 관리자를 사용합니다.

### 가상 환경(Virtual Environment) 사용하기 (권장)

프로젝트별로 독립된 Python 환경을 만들어주는 '가상 환경'을 사용하는 것이 좋습니다. 이렇게 하면 다른 Python 프로젝트와의 충돌을 막을 수 있습니다.

1.  **프로젝트 폴더 생성:** 분석 작업을 진행할 폴더를 하나 만듭니다. (예: `C:\volatility_workspace`)
2.  **가상 환경 생성:** 명령 프롬프트에서 해당 폴더로 이동한 뒤, 다음 명령어를 실행합니다.

    ```bash
    cd C:\volatility_workspace
    python -m venv venv
    ```
    `venv`라는 이름의 가상 환경이 생성됩니다.

3.  **가상 환경 활성화:**

    ```bash
    .\venv\Scripts\activate
    ```
    명령 프롬프트의 맨 앞에 `(venv)`라는 표시가 붙으면 가상 환경이 활성화된 것입니다.

### pip으로 Volatility3 설치

가상 환경이 활성화된 상태에서, 다음 명령어를 입력하여 Volatility3를 설치합니다.

```bash
pip install volatility3
```

설치가 완료되면, `vol.py -h` 명령어를 통해 도움말이 잘 출력되는지 확인해보세요.

```bash
# volatility3가 설치된 폴더의 volatility3 하위 폴더로 이동해야 할 수 있습니다.
# 예: cd .\venv\Lib\site-packages\volatility3
# 또는 전체 경로로 실행
# python C:\volatility_workspace\venv\Lib\site-packages\volatility3\vol.py -h

vol.py -h
```

> **팁:** `vol.py`를 편하게 사용하려면 `vol.py`가 있는 폴더(`C:\volatility_workspace\venv\Lib\site-packages\volatility3`)를 시스템 환경 변수 PATH에 추가하거나, `vol.py`의 바로 가기를 만들어 작업 폴더에 두면 편리합니다.

## 3. 실습용 메모리 덤프 준비

이제 분석할 '재료'인 메모리 덤프 파일이 필요합니다. 메모리 포렌식 학습 및 대회(CTF)에서는 주로 아래와 같은 공개된 메모리 덤프 파일을 많이 사용합니다.

- **Volatility Foundation Memory Samples:** Volatility 공식 재단에서 제공하는 샘플입니다. (웹 검색을 통해 `Volatility Foundation Memory Samples`를 찾아보세요.)
- **CTF 문제:** 각종 해킹 방어 대회나 워게임 사이트에서 포렌식 문제로 제공되는 메모리 덤프들이 좋은 실습 자료가 됩니다.

이 시리즈에서는 `memdump.mem`이라는 이름의 Windows 10 메모리 덤프 파일이 있다고 가정하고 설명을 진행하겠습니다. 여러분도 위와 같은 경로를 통해 실습에 사용할 파일을 다운로드하여 `C:\volatility_workspace` 폴더에 준비해주세요.

---

이제 모든 준비가 끝났습니다. Python 설치부터 Volatility3, 그리고 실습용 메모리 덤프까지 완벽한 분석 환경이 구축되었습니다.

다음 3부에서는 드디어 이 메모리 덤프 파일을 열어보고, 어떤 정보가 담겨있는지 확인하는 첫 분석 단계를 밟아보겠습니다. 또한 메모리 분석의 핵심 개념인 '심볼(Symbol)'에 대해서도 자세히 알아보겠습니다.
