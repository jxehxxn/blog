---
layout: post
title: "Kubernetes Package Management with Helm: Week 6 - CI/CD & Local Development (Skaffold)"
---

차트 개발은 피곤합니다.
1.  코드 수정
2.  도커 빌드 (`docker build ...`)
3.  도커 푸시 (`docker push ...`)
4.  Helm 업그레이드 (`helm upgrade ...`)
5.  확인

이 루틴을 하루에 100번 반복하면 퇴사하고 싶어집니다.
Google이 만든 **Skaffold**는 이 모든 과정을 **"코드 저장(Ctrl+S)"** 한 번으로 줄여줍니다.

오늘은 로컬 생산성을 10배 올려주는 **Skaffold**와, 차트의 품질을 보장하는 **Helm Test**를 배웁니다.

---

## 1. What is Skaffold?

Skaffold는 "Continuous Development for Kubernetes" 도구입니다.
파일 변경을 감지해서 -> 이미지 빌드 -> 태그 변경 -> Helm 배포까지 자동으로 해줍니다.

### `skaffold.yaml` 구조
```yaml
apiVersion: skaffold/v2beta29
kind: Config
build:
  artifacts:
    - image: my-app-image  # 이 이름의 이미지를
      context: .           # 현재 폴더의 Dockerfile로 빌드해라
deploy:
  helm:
    releases:
      - name: my-release
        chartPath: ./my-chart
        valuesFiles:
          - ./my-chart/values.yaml
        setValueTemplates:
          image.repository: "{{.IMAGE_REPO}}" # 빌드된 이미지 주소를 여기에 주입해라
          image.tag: "{{.IMAGE_TAG}}"
```

이제 터미널에서 `skaffold dev`만 켜두면 됩니다.

---

## 2. Helm Test (Hook)

"배포는 성공했는데(Pod Running), 앱이 제대로 동작하는지는 어떻게 알지?"
Helm에는 **Test Hook** 기능이 있습니다. `helm test` 명령어로 실행할 수 있는 일회성 Pod를 정의하는 것입니다.

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ .Release.Name }}-service:80']
  restartPolicy: Never
```

이 Pod가 **Exit Code 0**으로 죽으면 테스트 성공, 아니면 실패입니다.

---

## 3. Linting & Validation

차트를 배포하기 전에 문법 검사를 해야 합니다.

*   `helm lint ./my-chart`: YAML 문법, 필수 필드 체크.
*   `kubeval` or `pluto`: Deprecated API 사용 여부 체크. (Helm 플러그인으로 설치 가능)

---

## 🛠️ Lab: The Ultimate Dev Environment

Skaffold + Helm 조합을 구축해 봅니다.

1.  **애플리케이션:** 간단한 `server.js` (Node.js)와 `Dockerfile`을 준비합니다.
2.  **Helm Chart:** 이 앱을 배포하는 차트를 만듭니다.
3.  **Skaffold 설정:** `skaffold init`을 하면 자동으로 감지해서 `skaffold.yaml`을 만들어줍니다. (Deploy type을 helm으로 선택)
4.  **실행:** `skaffold dev`
5.  **검증:**
    *   `server.js`의 "Hello World"를 "Hello Skaffold"로 바꾸고 저장합니다.
    *   Skaffold가 자동으로 재빌드/재배포하는 로그를 지켜봅니다.
    *   브라우저에서 새로고침하면 바뀐 문구가 보입니다.

---

## 📝 6주차 과제: Integrated CI Pipeline

**목표:** GitHub Actions 워크플로우를 작성하여 차트 린팅과 테스트를 자동화하세요.

1.  `.github/workflows/ci.yaml` 작성.
2.  **Step 1:** Chart Testing Tool (`ct`) 설치.
3.  **Step 2:** `ct lint` 실행하여 차트 문법 검사.
4.  **Step 3:** (선택) Kind 클러스터 생성 후 `ct install` 실행하여 실제 설치 테스트.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Skaffold가 파일 변경을 감지했을 때 수행하는 3가지 주요 단계는? (Build -> Push -> Deploy)
2.  **Q2.** Helm 차트 내에서 `helm test` 명령어로 실행될 Pod를 정의하기 위해 달아야 하는 Annotation은? (`helm.sh/hook: test`)
3.  **Q3.** CI 파이프라인에서 Helm Chart의 변경 사항을 감지하고 린팅/테스트를 수행해주는 도구의 이름은? (Chart Testing - `ct`)

---

이것으로 Part 1: Helm 정복이 끝났습니다. 이제 여러분은 어떤 앱이든 패키징하고 배포할 수 있습니다.
다음 주부터는 Part 2: 관측 가능성(Observability)으로 넘어갑니다. 앱이 살았는지 죽었는지 감시하는 **Prometheus**와 **Grafana**를 배웁니다.

**Automate everything.**
