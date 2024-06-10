---
layout: post
title: "Splunk Week 14: Configuration as Code & DevSecOps"
date: 2024-05-31 09:00:00 +0900
categories: splunk
---

빅테크 기업에서 스플렁크를 운영할 때 가장 먼저 버리는 게 웹 UI 설정입니다. 수천 대의 서버에서 발생하는 로그를 처리하는 환경에서 마우스 클릭으로 알람을 만들고 대시보드를 수정하는 건 위험한 일이죠. 이번 14주차 수업에선 스플렁크 설정을 코드로 관리하는 방법과 이를 자동화하는 DevSecOps 파이프라인을 다룹니다.

왜 웹 UI를 피해야 할까요?
첫째, 변경 이력이 남지 않습니다. 누가 언제 어떤 의도로 SPL 쿼리를 수정했는지 알 길이 없죠. 사고가 터졌을 때 이전 상태로 되돌리는 일도 불가능에 가깝습니다. 둘째, 여러 환경 간의 동기화가 어렵습니다. 개발 환경에서 테스트한 대시보드를 운영 환경으로 옮길 때 복사 붙여넣기를 반복하다 보면 반드시 실수가 생깁니다.

스플렁크의 모든 구성 요소는 결국 파일입니다.
`savedsearches.conf`, `dashboards.xml`, `props.conf` 같은 설정 파일들이 스플렁크의 핵심입니다. 이걸 Git 저장소에 넣고 관리하면 그때부터 스플렁크는 단순한 도구가 아니라 소프트웨어가 됩니다. 코드 리뷰를 통해 보안 취약점을 점검하고, 테스트 코드를 돌려 쿼리 성능을 검증할 수 있습니다.

필수 도구들입니다.
1. Git: 버전 관리의 기본입니다. 모든 변경 사항을 기록합니다.
2. KSCONF: 스플렁크 설정 파일(.conf)을 병합하고 검증하는 전용 도구입니다. 여러 개발자가 같은 파일을 수정해도 충돌을 깔끔하게 해결해 줍니다.
3. Terraform: 스플렁크 인덱서나 서치 헤드 같은 인프라 자체를 코드로 배포할 때 사용합니다.

실제 워크플로우 예시를 보겠습니다.
보안 분석가가 새로운 이상 징후 탐지 알람을 추가한다고 가정해 봅시다.
1. 분석가는 로컬 환경에서 `savedsearches.conf` 파일에 새로운 SPL 쿼리를 작성합니다.
2. 작성한 코드를 새로운 브랜치에 푸시하고 Pull Request(PR)를 생성합니다.
3. 동료 분석가들이 쿼리의 효율성과 오탐 가능성을 리뷰합니다.
4. PR이 승인되면 메인 브랜치에 병합됩니다.
5. CI/CD 파이프라인이 자동으로 실행되어 KSCONF로 설정을 검증하고 서치 헤드 클러스터(SHC)에 배포합니다.

GitHub Actions 예시 코드입니다.
```yaml
name: Deploy Splunk Config
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install KSCONF
        run: pip install ksconf
      - name: Validate Config
        run: ksconf check ./my_splunk_app
      - name: Package App
        run: ksconf package ./my_splunk_app --file my_app.spl
      - name: Push to Search Head Cluster
        run: |
          # SHC Deployer API를 호출하여 앱 배포
          curl -k -u admin:${{ secrets.SPLUNK_PASSWORD }} \
          https://shc-deployer:8089/services/apps/local \
          -F "name=my_app" -F "appfile=@my_app.spl"
```

자가 진단 퀴즈입니다.
1. 스플렁크 설정을 코드로 관리할 때 얻는 가장 큰 이점은 무엇인가요?
2. KSCONF 도구의 주요 역할은 무엇인가요?
3. CI/CD 파이프라인에서 배포 전 반드시 수행해야 할 단계는 무엇인가요?

정답입니다.
1. 변경 이력 추적과 안정적인 롤백이 가능해집니다.
2. .conf 파일의 구문 검사, 병합, 패키징을 담당합니다.
3. KSCONF 등을 이용한 설정 파일의 구문 및 논리 검증 단계입니다.
