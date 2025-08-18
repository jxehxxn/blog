---
layout: post
title:  "Syft와 함께하는 소프트웨어 공급망 보안 첫걸음 (2/5) - 10분 만에 첫 SBOM 생성하기"
---

## 들어가며

지난 1부에서는 소프트웨어 공급망의 개념과 위험성, 그리고 그 해결책인 SBOM이 무엇인지 알아보았습니다. 이론의 중요성은 충분히 이해했으니, 이제 직접 우리 손으로 첫 번째 SBOM을 만들어 볼 시간입니다.

이번 2부에서는 여러분의 Windows PC에 Syft를 설치하고, 단 하나의 명령어로 프로젝트의 SBOM을 생성하는 과정을 차근차근 따라가 보겠습니다. 10분만 투자하면 여러분도 SBOM을 만들 수 있습니다!

## 1. Syft 설치하기 (Windows PowerShell)

Syft는 Windows, macOS, Linux 환경을 모두 지원합니다. 여기서는 Windows 사용자를 기준으로 PowerShell을 이용해 설치하는 가장 간편한 방법을 안내합니다.

#### 방법 1: PowerShell 스크립트로 자동 설치 (권장)

PowerShell을 관리자 권한으로 실행한 후, 아래 명령어를 복사하여 붙여넣고 실행해 주세요. 이 스크립트는 최신 버전의 Syft를 다운로드하여 바로 실행할 수 있도록 설정해 줍니다.

```powershell
powershell -c "[Net.ServicePointManager]::SecurityProtocol = 'Tls12'; iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/anchore/syft/main/install.ps1'))"
```

#### 방법 2: 수동 설치

자동 설치가 불안하거나 직접 통제하고 싶다면 수동으로 설치할 수도 있습니다.

1.  **Syft 릴리즈 페이지 접속:** [https://github.com/anchore/syft/releases](https://github.com/anchore/syft/releases) 로 이동합니다.
2.  **최신 버전 다운로드:** 가장 위에 있는 최신 릴리즈에서 `syft_..._windows_amd64.zip` 파일을 찾아 다운로드합니다.
3.  **압축 해제:** 다운로드한 zip 파일의 압축을 `C:\Program Files\Syft` 와 같은 원하는 폴더에 해제합니다. `syft.exe` 파일이 해당 폴더에 위치하게 됩니다.
4.  **환경 변수 PATH에 경로 추가:** 어느 위치에서든 `syft` 명령어를 실행할 수 있도록 `syft.exe`가 있는 폴더를 시스템 환경 변수 `Path`에 추가해야 합니다.
    *   `시스템 속성` -> `고급` -> `환경 변수` 로 이동합니다.
    *   `시스템 변수` 목록에서 `Path`를 선택하고 `편집`을 누릅니다.
    *   `새로 만들기`를 눌러 압축을 해제한 폴더 경로(예: `C:\Program Files\Syft`)를 추가하고 확인을 누릅니다.

## 2. 설치 확인

설치가 잘 되었는지 확인해 봅시다. PowerShell 창을 새로 열고 아래 명령어를 입력해 보세요.

```powershell
syft version
```

아래와 같이 Syft의 버전 정보가 표시된다면 성공적으로 설치된 것입니다.

```
Application:        syft
Version:            1.8.0
Json-Schema-Version: 8.0.1
BuildDate:          2025-08-15T18:12:01Z
GitCommit:          a1b2c3d4
GoVersion:          go1.22.5
Compiler:           gc
Platform:           windows/amd64
```
*(버전 정보는 설치 시점에 따라 다를 수 있습니다.)*

## 3. 분석할 프로젝트 준비

이제 SBOM을 생성할 샘플 프로젝트를 준비합시다. 여기서는 간단한 Node.js 웹 프레임워크인 Express.js의 공식 샘플 생성기를 사용하겠습니다. `git`이 설치되어 있어야 합니다.

PowerShell에서 다음 명령어를 실행하여 프로젝트를 복제(clone)하고 해당 폴더로 이동합니다.

```powershell
git clone https://github.com/expressjs/generator.git
cd generator
npm install
```

`npm install`은 이 프로젝트가 의존하는 모든 라이브러리(재료들)를 `node_modules` 폴더에 다운로드하는 과정입니다.

## 4. 첫 SBOM 생성!

드디어 첫 SBOM을 생성할 시간입니다. 방금 들어온 `generator` 폴더 안에서, 아래의 간단한 명령어를 실행해 보세요.

```powershell
syft .
```

`syft .`는 "현재 폴더(`.`)를 분석해서 SBOM을 만들어 줘" 라는 의미입니다. 잠시 후, 화면에 다음과 같은 표가 나타날 것입니다.

```
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [58 packages]

NAME              VERSION    TYPE
accepts           1.3.8      npm
ansi-styles       4.3.0      npm
array-flatten     1.1.1      npm
async             0.9.2      npm
balanced-match    1.0.2      npm
body-parser       1.19.2     npm
brace-expansion   1.1.11     npm
bytes             3.1.2      npm
chalk             4.1.2      npm
color-convert     2.0.1      npm
color-name        1.1.4      npm
commander         2.20.3     npm
concat-map        0.0.1      npm
content-disposition 0.5.4    npm
...
```

## 5. 결과 분석

축하합니다! 방금 여러분의 첫 SBOM을 성공적으로 생성했습니다. 표의 각 열이 무엇을 의미하는지 간단히 살펴볼까요?

- **NAME:** 소프트웨어 컴포넌트(라이브러리)의 이름입니다.
- **VERSION:** 해당 컴포넌트의 정확한 버전입니다. 버전 정보는 특정 취약점이 어떤 버전에 해당하는지 식별하는 데 매우 중요합니다.
- **TYPE:** 컴포넌트의 종류, 즉 어떤 생태계(ecosystem)에 속해 있는지를 나타냅니다. `npm`은 Node.js의 패키지임을 의미합니다. Python 프로젝트였다면 `python`, Go 프로젝트였다면 `go-module` 등으로 표시됩니다.

## 나가며

이번 포스트에서는 Syft를 설치하고 가장 기본적인 명령으로 SBOM을 생성하는 방법을 알아보았습니다. 생각보다 간단하지 않나요? 하지만 지금 본 것은 Syft가 가진 능력의 일부에 불과합니다. 기본 표 형식은 사람이 보기엔 편하지만, 기계가 읽고 다른 도구와 연동하기에는 부족합니다.

다음 3부, **Syft CLI 완벽 정복** 편에서는 오늘 실행한 `syft .` 명령에 다양한 옵션을 추가하여, 기계가 읽을 수 있는 표준 SBOM 포맷(SPDX, CycloneDX)으로 결과를 출력하고, 로컬 폴더가 아닌 Docker 컨테이너 이미지를 직접 분석하는 등 훨씬 더 강력하고 전문적인 Syft 활용법을 상세히 다뤄보겠습니다. 기대해 주세요!
