---
layout: post
title:  "Syft와 함께하는 소프트웨어 공급망 보안 첫걸음 (4/5) - Grype로 취약점 분석하기"
date:   2025-08-23 10:00:00 +0900
categories: Security DevSecOps SBOM Syft Grype Hands-on
---

## 들어가며

지난 3부까지 우리는 Syft의 전문가가 되어, 어떤 소스에서든 원하는 표준 형식의 SBOM을 생성하는 방법을 마스터했습니다. 이제 우리 손에는 소프트웨어를 구성하는 모든 재료가 담긴 상세한 '자재 명세서'가 들려 있습니다. 이제 가장 중요한 질문을 할 차례입니다. "그래서 이 목록으로 무엇을 할 수 있을까?"

이번 4부에서는 이 질문에 대한 답을 찾아갑니다. 우리가 만든 SBOM을 활용하여, 그 안에 포함된 라이브러리들이 가지고 있는 잠재적인 보안 위협, 즉 **취약점(Vulnerability)**을 찾아내는 방법을 알아보겠습니다. 이 여정에는 Syft의 강력한 자매 도구인 **Grype**가 함께합니다.

## 1. 취약점이란? 그리고 CVE

먼저 **취약점**이란, 공격자가 시스템의 보안을 해치기 위해 악용할 수 있는 소프트웨어의 설계상 결함이나 코딩 오류를 의미합니다. 그리고 전 세계적으로 발견되는 수많은 취약점들을 체계적으로 관리하기 위해 고유한 ID를 부여하는데, 이것이 바로 **CVE(Common Vulnerabilities and Exposures)** 입니다. (예: `CVE-2021-44228`은 Log4j의 치명적 취약점에 부여된 ID입니다.)

우리의 목표는 우리 SBOM에 있는 수많은 라이브러리들 중, 알려진 CVE를 가진 버전이 포함되어 있는지 찾아내는 것입니다.

## 2. Grype: SBOM을 위한 취약점 스캐너

**Grype**는 Syft와 마찬가지로 Anchore사에서 만든 오픈소스 도구로, 소프트웨어 컴포넌트 목록을 받아 알려진 취약점 데이터베이스와 비교하여 보안 위협을 찾아주는 스캐너입니다. Syft가 '무엇이 들어있는지' 알려준다면, Grype는 '그 안에 위험한 것은 없는지' 알려주는 역할을 합니다.

### Grype 설치하기

설치는 Syft와 거의 동일합니다. PowerShell을 관리자 권한으로 실행하고 아래 명령어를 입력하세요.

```powershell
powershell -c "[Net.ServicePointManager]::SecurityProtocol = 'Tls12'; iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/anchore/grype/main/install.ps1'))"
```

설치 후, PowerShell을 새로 열어 `grype version` 명령으로 설치를 확인합니다.

## 3. 취약점 스캔 실행하기

2부에서 사용했던 `expressjs/generator` 프로젝트 폴더에서 스캔을 진행해 보겠습니다.

### 방법 1: Grype로 직접 스캔하기

Grype는 Syft처럼 자체적으로 프로젝트를 분석하는 기능도 내장하고 있습니다. 가장 간단하게 취약점을 찾는 방법입니다.

```powershell
grype .
```

잠시 후 Grype가 취약점 데이터베이스를 업데이트하고 스캔을 마친 뒤, 다음과 같은 결과를 보여줄 것입니다.

```
 ✔ Vulnerability DB        [updated]
 ✔ Scanned image           [58 packages]
 ✔ Found 12 vulnerabilities

NAME              INSTALLED    FIXED-IN          VULNERABILITY     SEVERITY
ejs               3.1.7        3.1.8             CVE-2022-29078    Critical
minimist          1.2.5        1.2.6             CVE-2021-44906    High
async             0.9.2                          GHSA-fwr7-v2mv-xvcq Medium
...
```

### 방법 2: Syft의 SBOM과 연동하기 (권장)

더욱 강력하고 유연한 방법은 Syft로 SBOM을 생성하고, 그 결과를 파이프(`|`)로 Grype에 바로 전달하는 것입니다. 이는 '하나의 도구는 하나의 역할을 잘한다'는 유닉스 철학과도 맞닿아 있습니다.

> **Syft (SBOM 생성)** -> **Grype (취약점 분석)**

```powershell
syft . -o cyclonedx-json | grype
```

이 명령은 Syft에게 CycloneDX 형식의 SBOM을 생성하여 화면에 출력하라고 지시하고, 파이프(`|`)는 그 출력 결과를 그대로 Grype의 입력으로 전달합니다. Grype는 파일이나 폴더 대신 SBOM 데이터를 직접 받아 분석을 시작합니다. 결과는 방법 1과 동일하지만, 이 방식은 SBOM 생성을 명시적으로 제어할 수 있다는 장점이 있습니다.

## 4. 스캔 결과 심층 분석

결과 테이블의 각 열은 우리에게 중요한 정보를 알려줍니다.

- **NAME:** 취약점이 발견된 컴포넌트의 이름입니다.
- **INSTALLED:** 현재 설치된 버전입니다.
- **FIXED-IN:** 취약점이 해결된 버전입니다. **이것이 가장 중요한 정보입니다!** 개발자는 이 정보를 보고 해당 라이브러리를 어떤 버전으로 업데이트해야 할지 명확히 알 수 있습니다.
- **VULNERABILITY:** 취약점의 고유 ID (CVE 또는 GHSA 등) 입니다. 이 ID로 구글에 검색하면 해당 취약점에 대한 상세 정보를 얻을 수 있습니다.
- **SEVERITY:** 취약점의 심각도입니다. `Critical`, `High`, `Medium`, `Low`, `Negligible` 등으로 나뉘며, 일반적으로 `Critical`이나 `High` 등급의 취약점은 즉시 조치해야 합니다.

예를 들어, 위 결과의 첫 줄은 `ejs` 라이브러리 `3.1.7` 버전에 `Critical` 등급의 `CVE-2022-29078` 취약점이 있으며, `3.1.8` 버전으로 업데이트하면 해결된다는 아주 명확하고 실행 가능한 정보를 제공합니다.

## 나가며

드디어 우리는 소프트웨어 공급망 보안의 핵심 사이클을 완성했습니다. Syft로 내 코드의 구성 요소를 파악하여 SBOM을 만들고, Grype로 그 SBOM을 분석하여 실제적인 보안 위협을 찾아내고 해결 방향까지 확인했습니다. 이제 여러분은 더 이상 "내 코드에 뭐가 들어있는지 모르겠어요"라고 말하지 않고, "내 코드의 `ejs` 라이브러리를 `3.1.8`로 업데이트해야 합니다"라고 자신 있게 말할 수 있게 되었습니다.

이제 마지막 단계만 남았습니다. 이 모든 유용한 과정을 어떻게 하면 잊어버리지 않고, 매일매일, 모든 코드 변경에 대해 자동으로 수행할 수 있을까요? 마지막 5부에서는 GitHub Actions를 이용해 이 모든 과정을 자동화하여, 숨 쉬는 것처럼 자연스럽게 보안을 실천하는 DevSecOps 파이프라인을 구축하는 방법을 알아보겠습니다.
