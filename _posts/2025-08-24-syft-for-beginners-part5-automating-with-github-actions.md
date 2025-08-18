---
layout: post
title:  "Syft와 함께하는 소프트웨어 공급망 보안 첫걸음 (5/5) - GitHub Actions로 자동화하기"
date:   2025-08-24 10:00:00 +0900
categories: Security DevSecOps SBOM Syft Grype GitHub-Actions Hands-on
---

## 들어가며: 사람의 기억을 믿지 마세요

지난 4부까지, 우리는 Syft와 Grype를 사용하여 수동으로 SBOM을 생성하고 취약점을 분석하는 강력한 방법을 배웠습니다. 하지만 이 과정이 아무리 강력하더라도, 사람이 매번 기억해서 실행해야 한다면 언젠가는 실수가 발생하고, 결국 보안 검사는 잊혀지게 될 것입니다. 진정한 보안은 일회성 이벤트가 아니라, 자동화되고 일상적인 프로세스일 때 완성됩니다.

시리즈의 마지막인 이번 5부에서는, 우리가 배운 모든 것을 **GitHub Actions**라는 CI/CD 도구를 통해 자동화하는 방법을 알아보겠습니다. 이를 통해, 코드가 변경될 때마다 로봇이 자동으로 SBOM을 생성하고 취약점을 검사하여 우리 저장소의 보안을 24시간 지키도록 만들 것입니다. 이것이 바로 **DevSecOps(Development + Security + Operations)**의 핵심입니다.

## 1. CI/CD와 GitHub Actions란?

- **CI/CD (Continuous Integration/Continuous Deployment)**: 개발자가 코드를 변경했을 때, 그 코드를 자동으로 빌드, 테스트하고 배포하는 전체 과정을 의미합니다. CI/CD는 개발의 속도와 안정성을 크게 향상시킵니다.
- **GitHub Actions**: GitHub에 내장된 CI/CD 서비스입니다. 우리 저장소에 특정 이벤트(예: `push`, `pull_request`)가 발생했을 때, 미리 정의된 작업(Workflow)을 자동으로 실행해 줍니다.

우리의 목표는, `main` 브랜치에 코드가 `push` 되거나 `pull_request`가 생성될 때마다 Syft와 Grype를 자동으로 실행하는 워크플로우를 만드는 것입니다.

## 2. GitHub Actions 워크플로우 만들기

먼저, 우리 프로젝트 루트에 `.github/workflows` 라는 폴더를 만들어야 합니다. GitHub Actions는 이 폴더 안에 있는 `.yml` 또는 `.yaml` 파일을 워크플로우로 인식합니다.

그 안에 `supply-chain-security.yml` 라는 이름으로 새 파일을 생성합시다.

**파일 경로:** `.github/workflows/supply-chain-security.yml`

이제 이 파일에 워크플로우의 내용을 작성해 보겠습니다.

## 3. 워크플로우 파일 상세 작성

아래의 전체 코드를 `supply-chain-security.yml` 파일에 복사하여 붙여넣으세요. 각 섹션에 대한 설명은 코드 아래에 이어집니다.

```yaml
name: Software Supply Chain Security Scan

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  scan:
    name: Syft and Grype Scan
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm install

      - name: Generate SBOM
        id: syft
        uses: anchore/syft-action@v1
        with:
          format: spdx-json
          output-file: "./sbom.spdx.json"

      - name: Scan for vulnerabilities
        uses: anchore/grype-action@v1
        with:
          sbom: "./sbom.spdx.json"
          fail-on-severity: high

      - name: Upload SBOM as an artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom-spdx
          path: ./sbom.spdx.json
```

- **`name`**: 이 워크플로우의 이름입니다.
- **`on`**: 언제 이 워크플로우를 실행할지 정의합니다. `main` 브랜치에 `push` 되거나 `pull_request`가 올 때 실행됩니다.
- **`jobs`**: 워크플로우가 수행할 작업들의 목록입니다. 우리는 `scan`이라는 하나의 job을 정의했습니다.
- **`runs-on`**: 어떤 가상 환경에서 job을 실행할지 지정합니다. `ubuntu-latest`를 사용합니다.
- **`steps`**: job이 수행할 단계별 명령어 목록입니다.
    1.  **`actions/checkout@v4`**: 저장소의 코드를 가상 환경으로 가져옵니다.
    2.  **`actions/setup-node@v4`**: `npm install`을 위해 Node.js 환경을 설정합니다.
    3.  **`Install dependencies`**: `npm install`을 실행하여 라이브러리를 설치합니다.
    4.  **`Generate SBOM`**: 공식 `syft-action`을 사용하여 `spdx-json` 형식의 SBOM을 `sbom.spdx.json` 파일로 생성합니다.
    5.  **`Scan for vulnerabilities`**: 공식 `grype-action`을 사용하여 바로 전 단계에서 만든 SBOM 파일을 스캔합니다. **`fail-on-severity: high`** 옵션이 핵심입니다. 이 옵션은 심각도가 `high` 이상인 취약점이 발견되면 워크플로우를 **실패** 처리하여, 문제가 있는 코드가 병합되는 것을 막는 '보안 게이트' 역할을 합니다.
    6.  **`Upload SBOM`**: 생성된 SBOM 파일을 `artifact`(결과물)로 업로드하여, 나중에 우리가 다운로드하여 확인할 수 있게 합니다.

## 4. 결과 확인하기

이 파일을 저장소에 추가하고 GitHub에 푸시하면, 저장소의 **Actions** 탭에서 워크플로우가 실행되는 것을 볼 수 있습니다. Job이 성공적으로 끝나면 녹색 체크 표시가 나타납니다. 만약 Grype가 `high` 등급 이상의 취약점을 발견했다면, Job은 실패하고 빨간색 X 표시가 나타날 것입니다. 이것은 문제가 있다는 신호이며, 개발자는 Pull Request를 병합하기 전에 해당 취약점을 먼저 해결해야 합니다.

## 시리즈를 마치며

지난 5부작에 걸쳐 우리는 소프트웨어 공급망 보안이라는, 다소 막연하게 들렸던 개념에서부터 출발했습니다. SBOM의 필요성을 이해하고(1부), Syft를 설치하여 첫 SBOM을 만들었으며(2부), 전문가처럼 Syft의 모든 기능을 파고들었습니다(3부). 그리고 Grype를 이용해 SBOM에서 실제 보안 위협을 찾아내는 실용적인 방법을 배웠고(4부), 마침내 오늘 이 모든 과정을 자동화하여 살아 숨 쉬는 보안 파이프라인을 구축했습니다(5부).

이제 여러분은 더 이상 수동적이고 방어적인 보안에 머무르지 않고, 개발의 모든 단계에 보안을 통합하는 능동적인 DevSecOps를 실천할 수 있는 강력한 무기를 갖게 되었습니다. 여러분의 프로젝트에 오늘 배운 워크플로우를 적용하여, 더 안전하고 신뢰할 수 있는 소프트웨어를 만들어 나가시길 바랍니다.

그동안 함께해 주셔서 감사합니다!
