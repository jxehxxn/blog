---
layout: post
title:  "Syft와 함께하는 소프트웨어 공급망 보안 첫걸음 (3/5) - Syft CLI 완벽 정복"
date:   2025-08-22 10:00:00 +0900
categories: Security DevSecOps SBOM Syft Hands-on
---

## 들어가며

지난 2부에서는 `syft .` 라는 간단한 명령어로 첫 SBOM을 생성해 보았습니다. 하지만 이 기본 테이블 출력은 사람이 보기에는 편하지만, 자동화된 시스템이나 다른 보안 도구와 연동하기에는 한계가 있습니다. Syft의 진정한 힘은 강력하고 유연한 CLI(Command-Line Interface) 옵션에서 나옵니다.

이번 3부에서는 여러분을 Syft 초보자에서 전문가로 만들어 줄 다양한 CLI 기능과 설정 파일 사용법을 상세히 파고들어 보겠습니다. 이 가이드를 마치고 나면, 어떤 상황에서든 원하는 형태로 SBOM을 생성하고 관리할 수 있게 될 것입니다.

## 1. 출력 포맷 제어 (`-o`, `--output`): SBOM의 표준 언어

SBOM의 핵심은 '기계가 읽을 수 있는(machine-readable)' 정형화된 형식입니다. Syft는 `-o` 또는 `--output` 플래그를 통해 다양한 표준 포맷을 지원합니다.

#### 주요 출력 포맷

- **`table`**: (기본값) 사람이 보기 좋은 표 형식입니다.
- **`syft-json`**: Syft가 수집한 모든 정보를 담고 있는 상세한 JSON 형식입니다. 다른 도구와 연동하기보다는, Syft 자체의 전체 결과를 보고 싶을 때 유용합니다.
- **`spdx-json` / `spdx-xml`**: Linux Foundation의 표준인 **SPDX(Software Package Data Exchange)** 형식입니다. 가장 널리 쓰이는 SBOM 표준 중 하나입니다.
- **`cyclonedx-json` / `cyclonedx-xml`**: OWASP에서 주도하는 보안 중심의 SBOM 표준 **CycloneDX** 형식입니다. 특히 보안 취약점 정보와 함께 사용될 때 강력합니다.
- **`github-json`**: GitHub의 Dependency Submission API와 호환되는 형식입니다. GitHub Actions에서 이를 사용하면 저장소의 'Dependency graph'를 자동으로 채울 수 있습니다.

#### 사용 예시

SPDX 형식의 JSON 파일로 결과를 저장하고 싶다면 다음과 같이 명령합니다.

```powershell
# spdx-json 형식으로 화면에 출력
syft . -o spdx-json

# spdx-json 형식으로 파일에 저장
syft . -o spdx-json > sbom.spdx.json
```

## 2. 스캔 대상 지정: 내 컴퓨터 너머의 세상

Syft는 현재 폴더 외에도 다양한 소스를 직접 분석할 수 있습니다. 각 소스는 `[source]:[path]` 형식으로 지정합니다.

- **`dir:[path]`**: 특정 디렉터리를 지정하여 분석합니다.
  ```powershell
  syft dir:C:/Users/note/Desktop/dev/project
  ```

- **`image:[name]`**: 로컬에 다운로드된 컨테이너 이미지를 분석합니다. 이는 컨테이너 보안의 첫걸음입니다.
  ```powershell
  # 내 컴퓨터의 Docker에 있는 my-app:1.0 이미지를 분석
  syft image:my-app:1.0
  ```

- **`registry:[name]`**: 원격 컨테이너 레지스트리(예: Docker Hub)에서 이미지를 바로 다운로드하여 분석합니다. 로컬에 이미지가 없어도 됩니다.
  ```powershell
  # Docker Hub의 최신 alpine 이미지를 분석
  syft registry:alpine:latest
  ```

- **기타**: `tar:[path]` (tar 아카이브), `podman:[name]` (Podman 이미지) 등 다양한 소스를 지원합니다.

## 3. 스캔 범위 정밀 제어

때로는 스캔 과정에서 특정 파일이나 폴더를 제외하거나, 컨테이너 이미지의 특정 레이어만 보고 싶을 수 있습니다.

- **`--exclude [path]`**: 특정 경로를 분석에서 제외합니다. 여러 번 사용하여 다수의 경로를 제외할 수 있습니다.
  ```powershell
  # 테스트 코드와 빌드 결과물은 제외하고 분석
  syft . --exclude "**/*_test.go" --exclude "./dist"
  ```

- **`--scope [scope]`**: 컨테이너 이미지 분석 시 범위를 지정합니다. `Squashed` (기본값, 최종 압축된 파일 시스템) 또는 `AllLayers` (모든 레이어)를 선택할 수 있습니다.
  ```powershell
  # alpine 이미지의 모든 레이어를 개별적으로 분석
  syft alpine:latest --scope AllLayers
  ```

- **`--platform [platform]`**: 다중 아키텍처를 지원하는 이미지의 경우, 특정 플랫폼(예: `linux/arm64`)을 지정하여 분석할 수 있습니다.

## 4. 설정 파일 (`.syft.yaml`)로 나만의 기본값 만들기

매번 긴 옵션을 입력하는 것은 번거롭습니다. 프로젝트 루트 디렉터리에 `.syft.yaml` 파일을 만들어 두면, `syft` 명령 실행 시 이 파일을 자동으로 읽어 기본 설정으로 사용합니다.

#### `.syft.yaml` 예시

프로젝트 루트에 아래 내용으로 `.syft.yaml` 파일을 만들어 보세요.

```yaml
# 기본 출력 형식을 CycloneDX JSON으로 설정
output: cyclonedx-json

# SBOM 결과를 항상 이 파일 이름으로 저장
file: sbom.cdx.json

# 분석에서 제외할 경로들
exclude:
  - "**/*.test.js"
  - "**/*.spec.js"
  - "dist"
  - "node_modules/jest"

# 조용한 모드 (로고나 진행률 표시 안 함)
quiet: true
```

이제 이 폴더에서 `syft` 라고만 입력해도, 위 설정이 모두 적용되어 `sbom.cdx.json` 파일이 조용히 생성됩니다. 훨씬 간편하죠?

## 나가며

이번 시간에는 Syft의 강력한 CLI 옵션과 설정 파일을 통해 SBOM 생성을 자유자재로 제어하는 방법을 깊이 있게 알아보았습니다. 이제 여러분은 어떤 프로젝트나 컨테이너 이미지를 만나더라도, 원하는 표준 형식과 내용으로 SBOM을 자신 있게 생성할 수 있는 능력을 갖추게 되었습니다.

하지만 SBOM은 그 자체로 끝이 아닙니다. 진짜 중요한 것은 '그래서 이 재료 목록으로 무엇을 할 것인가?' 입니다. 다음 4부에서는 우리가 만든 SBOM을 활용하여 그 안에 숨어있는 보안 취약점을 찾아내는 실질적인 보안 분석 단계로 넘어가 보겠습니다. 자매 도구인 `Grype`와 함께하는 여정을 기대해 주세요!
