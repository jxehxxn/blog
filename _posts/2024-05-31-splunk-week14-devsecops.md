---
layout: post
title: "Splunk Week 14: Configuration as Code & DevSecOps Deep Dive"
date: 2024-05-31 09:00:00 +0900
categories: splunk
---

# 스플렁크 DevSecOps: 설정의 코드화와 자동화 파이프라인 구축

빅테크 기업이나 대규모 엔터프라이즈 환경에서 스플렁크를 운영할 때 가장 먼저 직면하는 한계는 웹 UI 기반의 관리 방식입니다. 수천 대의 서버에서 발생하는 테라바이트급 로그를 처리하는 환경에서 마우스 클릭으로 알람을 만들고 대시보드를 수정하는 'Click-Ops'는 운영의 병목이자 보안 사고의 씨앗이 됩니다.

이번 14주차 강의에선 스플렁크 설정을 코드로 관리하는 Configuration as Code(CaC)의 철학과 이를 실제 운영 환경에 적용하기 위한 DevSecOps 파이프라인 구축 방법을 심도 있게 다룹니다.

---

## 1. 왜 Click-Ops를 버리고 GitOps로 가야 하는가?

전통적인 스플렁크 운영 방식에선 관리자가 웹 UI에서 직접 설정을 변경합니다. 하지만 이 방식은 다음과 같은 치명적인 결함을 가집니다.

1. **변경 이력의 부재**: 누가, 언제, 왜 SPL 쿼리를 수정했는지 알 수 없습니다. 사고 발생 시 이전 상태로의 롤백이 불가능에 가깝습니다.
2. **환경 간 불일치**: 개발(Dev), 검증(Staging), 운영(Prod) 환경 간의 설정을 동기화하기 어렵습니다. 수동 복사 과정에서 반드시 실수가 발생합니다.
3. **검증되지 않은 쿼리**: 비효율적인 SPL 쿼리가 운영 환경에 배포되어 서치 헤드의 성능을 저하시키거나 인덱서에 과부하를 줄 수 있습니다.
4. **보안 취약점**: 민감한 정보가 포함된 설정이 그대로 노출되거나, 권한 관리가 제대로 이루어지지 않은 상태로 배포될 위험이 있습니다.

GitOps는 Git 저장소를 '단일 진실 공급원(Single Source of Truth)'으로 삼아 모든 설정을 코드로 관리하고, 자동화된 파이프라인을 통해 배포하는 방식입니다. 이를 통해 운영의 투명성, 안정성, 보안성을 동시에 확보할 수 있습니다.

---

## 2. 스플렁크 설정 생태계의 이해

스플렁크의 모든 구성 요소는 결국 파일입니다. `.conf`, `.xml`, `.py` 파일들이 모여 하나의 앱(App)을 구성합니다.

### 설정 우선순위 (Precedence)
스플렁크는 여러 위치의 설정을 병합하여 최종 설정을 결정합니다.
- **System local** > **App local** > **App default** > **System default**
- **User** 설정은 가장 높은 우선순위를 가집니다.

### Default vs Local 딜레마
Git으로 설정을 관리할 때 가장 중요한 원칙은 **"Local 디렉토리를 직접 수정하지 않는다"**는 것입니다.
- **Default**: 앱의 기본 설정이 들어갑니다. Git 저장소에서 관리되는 영역입니다.
- **Local**: 웹 UI에서 변경한 사항이 저장되는 곳입니다. GitOps 환경에선 파이프라인이 배포한 설정이 Local을 덮어쓰거나, 아예 Local 사용을 금지해야 합니다.

---

## 3. 필수 도구 세트 (The Tooling Stack)

### Git
모든 설정의 버전 관리를 담당합니다. 코드 리뷰와 승인 절차를 통해 변경 사항을 통제합니다.

### KSCONF (Kintyre Splunk Configuration Tool)
스플렁크 설정 파일 전용 스위스 아미 나이프입니다.
- **Check**: `.conf` 파일의 구문 오류를 검사합니다.
- **Combine**: 여러 소스의 설정을 하나로 병합합니다.
- **Package**: 배포 가능한 `.spl` 파일을 생성합니다.
- **Promote**: Local의 변경 사항을 Default로 안전하게 옮깁니다.

### Splunk-appinspect
스플렁크 공식 앱 검증 도구입니다. 보안 취약점, 성능 이슈, 클라우드 호환성 등을 자동으로 체크합니다.

---

## 4. Git 워크플로우와 브랜칭 전략

스플렁크 지식 객체(Knowledge Objects)를 관리하기 위한 최적의 전략은 **Trunk-based Development** 또는 **GitFlow**의 변형입니다.

### 브랜칭 구조
1. **feature/**: 새로운 알람, 대시보드, 매크로 등을 개발하는 브랜치입니다.
2. **develop**: 개발 환경에 자동으로 배포되어 통합 테스트를 거치는 브랜치입니다.
3. **main (or master)**: 운영 환경에 배포되는 최종 승인된 코드입니다.

### 지식 객체 관리 전략
- **App 단위 관리**: 각 스플렁크 앱을 별도의 Git 저장소나 모노레포의 디렉토리로 관리합니다.
- **환경별 변수 처리**: 인덱스 이름이나 서버 주소 등 환경마다 다른 값은 KSCONF의 `combine` 기능을 이용해 환경별 레이어를 겹쳐 해결합니다.

---

## 5. DevSecOps 파이프라인 구축

자동화 파이프라인은 크게 CI(지속적 통합)와 CD(지속적 배포) 단계로 나뉩니다.

### CI 단계: 검증과 테스트
1. **SPL Linting**: 정규표현식을 이용해 `join`, `append` 등 성능 저하 유발 명령어나 와일드카드(`*`) 남용을 체크합니다.
2. **Config Validation**: `ksconf check`로 설정 파일의 문법을 검사합니다.
3. **AppInspect**: 보안 및 표준 준수 여부를 확인합니다.

### CD 단계: 패키징과 배포
1. **Packaging**: 검증된 코드를 `.spl` 파일로 묶습니다.
2. **Deployment**: 
    - **Search Head Cluster (SHC)**: Deployer의 REST API를 호출하여 배포합니다.
    - **Indexer Cluster (IDXC)**: Cluster Master(Manager)를 통해 배포합니다.
    - **Standalone/DS**: Deployment Server를 통해 배포합니다.

---

## 6. 실전: GitHub Actions YAML 구현

다음은 SPL 린팅, KSCONF 검증, 그리고 SHC Deployer로의 자동 배포를 포함하는 전체 파이프라인 예시입니다.

```yaml
name: Splunk DevSecOps Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install Tools
        run: |
          pip install ksconf splunk-appinspect

      - name: KSCONF Syntax Check
        run: |
          ksconf check ./my_splunk_app/default/*.conf

      - name: SPL Performance Linting
        run: |
          # join이나 append 사용 시 경고 발생 (예시 스크립트)
          grep -rnE "join|append" ./my_splunk_app/default/savedsearches.conf && echo "Warning: Inefficient SPL detected" || echo "SPL looks good"

      - name: Splunk AppInspect
        run: |
          splunk-appinspect inspect ./my_splunk_app --mode precert

  deploy:
    needs: validate
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop')
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Install KSCONF
        run: pip install ksconf

      - name: Package App
        run: |
          ksconf package ./my_splunk_app --file my_app.spl --app-name my_app

      - name: Deploy to SHC Deployer
        env:
          SPLUNK_HOST: ${{ github.ref == 'refs/heads/main' && secrets.PROD_DEPLOYER || secrets.DEV_DEPLOYER }}
          SPLUNK_PASS: ${{ secrets.SPLUNK_PASSWORD }}
        run: |
          # SHC Deployer API를 통한 앱 업로드 및 배포
          # 실제 환경에선 인증서 검증(-k) 주의
          curl -k -u admin:$SPLUNK_PASS \
          https://$SPLUNK_HOST:8089/services/apps/local \
          -F "name=my_app" -F "appfile=@my_app.spl" -F "update=true"
          
          # 배포 적용 (Apply Bundle)
          curl -k -u admin:$SPLUNK_PASS -X POST \
          https://$SPLUNK_HOST:8089/services/apps/deployments/shcluster/member/control/control/apply
```

---

## 7. 보안과 거버넌스 (Security & Governance)

### 비밀번호 관리 (Secret Management)
파이프라인에서 사용하는 스플렁크 계정 정보는 절대 코드에 노출되어선 안 됩니다. GitHub Secrets, HashiCorp Vault 등을 연동하여 관리해야 합니다.

### 감사 추적 (Audit Trail)
Git의 커밋 로그는 그 자체로 훌륭한 감사 자료가 됩니다. 누가 어떤 알람의 임계치를 수정했는지, 어떤 대시보드에 새로운 패널을 추가했는지 명확히 기록됩니다.

### 정책 기반 배포 (Policy-based Deployment)
특정 태그가 붙은 커밋만 운영 환경에 배포되도록 제한하거나, 반드시 두 명 이상의 리뷰어 승인이 있어야 병합되도록 설정하여 거버넌스를 강화합니다.

---

## 8. 결론: 스플렁크 운영의 미래

스플렁크 DevSecOps는 단순히 자동화를 넘어 운영의 패러다임을 바꾸는 일입니다. 설정을 코드로 관리함으로써 얻는 안정성과 속도는 보안 팀이 위협 대응에 더 집중할 수 있는 환경을 만들어 줍니다.

처음에는 파이프라인 구축이 번거롭게 느껴질 수 있습니다. 하지만 한 번 구축된 시스템은 수동 운영에서 발생하는 수많은 장애와 보안 사고를 막아주는 든든한 방패가 될 것입니다. 지금 바로 여러분의 스플렁크 환경에 Git을 도입해 보세요.

---

## 자가 진단 퀴즈

1. **스플렁크 설정 우선순위에서 `App local`과 `App default` 중 어느 것이 더 높은가요?**
   - 정답: `App local`이 더 높습니다.

2. **KSCONF의 `combine` 기능이 GitOps 환경에서 중요한 이유는 무엇인가요?**
   - 정답: 여러 소스(기본 설정, 환경별 설정 등)를 병합하여 최종 배포용 설정을 생성할 수 있기 때문입니다.

3. **CI 파이프라인에서 SPL 린팅을 수행하는 주된 목적은 무엇인가요?**
   - 정답: 비효율적인 쿼리를 사전에 차단하여 운영 환경의 성능 저하를 방지하기 위함입니다.
