---
layout: post
title: "Kubernetes Package Management with Helm: Week 3 - Chart Development Basics"
---

남이 만든 차트를 쓰는 건 쉽습니다. 하지만 실무에서는 우리 회사의 비즈니스 로직이 담긴 애플리케이션을 배포해야 합니다. 즉, **Custom Chart**를 만들어야 합니다.

"그냥 `helm create` 하면 다 만들어주던데요?"
네, 하지만 그건 범용 템플릿입니다. 불필요한 게 너무 많고, 정작 우리에게 필요한 건 없을 수도 있습니다. 오늘은 `helm create`가 만들어준 파일들을 싹 지우고, **백지(Scratch)에서부터 하나씩 쌓아올리며** 원리를 깨우칩니다.

---

## 1. The `Chart.yaml`: The ID Card

모든 차트의 시작점입니다. 필수 필드를 알아봅시다.

```yaml
apiVersion: v2
name: my-app        # 차트 이름 (필수)
description: My Web Application
type: application   # application 또는 library
version: 0.1.0      # 차트의 버전 (SemVer 준수 필수!)
appVersion: "1.16.0" # 내 앱(이미지)의 버전
```

**Tip:** `version`을 올리지 않고 레포지토리에 푸시하면 덮어씌워지거나 에러가 납니다. 배포할 때마다 버전을 올리는 습관을 들이세요.

---

## 2. `values.yaml`: The Interface

`values.yaml`은 차트 개발자와 사용자 사이의 **약속(Interface)**입니다.
사용자는 템플릿 코드를 몰라도, 이 파일만 보고 설정을 바꿀 수 있어야 합니다.

**Bad Practice:**
```yaml
# 너무 납작한 구조
imageName: nginx
imageTag: latest
servicePort: 80
replica: 1
```

**Good Practice (Structured):**
```yaml
# 계층적인 구조
image:
  repository: nginx
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

replicaCount: 1
```

이렇게루핑(Looping)이나 객체화하기 좋게 구조를 잡아야 합니다.

---

## 3. Templates: The Engine

이제 `templates/` 폴더 안에 YAML 파일을 만듭니다.

### String Injection
가장 기본적인 문법입니다.
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deploy
```
*   `.Release.Name`: 사용자가 `helm install` 할 때 지어준 이름.
*   `.Values.foo`: `values.yaml`에 있는 `foo` 값.

### Pipeline & Functions
데이터를 가공할 때 씁니다.
```yaml
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" | quote }}
```
*   `default "latest"`: 태그 값이 없으면 "latest"를 써라.
*   `quote`: 문자열에 쌍따옴표를 씌워라. (YAML 파싱 에러 방지용)

---

## 🛠️ Lab: "Hello World" Chart

아주 간단한 Nginx 차트를 직접 만들어봅니다.

### 1. 차트 생성
```bash
mkdir my-chart
cd my-chart
touch Chart.yaml values.yaml
mkdir templates
```

### 2. Chart.yaml 작성
위에서 배운 내용을 참고해 필수 필드를 채우세요.

### 3. values.yaml 작성
```yaml
replicaCount: 2
image: nginx:alpine
message: "Hello from Helm"
```

### 4. ConfigMap 템플릿 작성 (`templates/configmap.yaml`)
Nginx의 index.html을 교체하는 ConfigMap을 만듭니다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-index
data:
  index.html: |
    <h1>{{ .Values.message }}</h1>
```

### 5. Deployment 템플릿 작성 (`templates/deployment.yaml`)
ConfigMap을 볼륨으로 마운트하는 Deployment를 작성합니다. (Pod 스펙은 간단하게)

### 6. 테스트 (Dry Run)
실제 설치 전에 렌더링된 결과를 확인합니다. (매우 중요!)
```bash
helm install --dry-run --debug test-release ./
```

---

## 📝 3주차 과제: Parameterize Everything

Lab에서 만든 차트를 발전시키세요.

1.  **Service 추가:** `templates/service.yaml`을 만들고 `NodePort`로 노출하세요. 포트 번호는 `values.yaml`에서 가져와야 합니다.
2.  **Resource 제한:** Pod의 CPU/Memory 요청량(requests/limits)을 `values.yaml`에서 설정할 수 있게 만드세요.
3.  **Labeling:** 모든 리소스(Deployment, Service, ConfigMap)에 공통 Label (`app: {{ .Release.Name }}`)을 붙이세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Helm 템플릿에서 `.Release`, `.Values`, `.Chart` 같은 최상위 객체에 접근하기 위해 항상 맨 앞에 붙여야 하는 기호는? (점 `.`)
2.  **Q2.** `values.yaml`에 정의되지 않은 값을 템플릿에서 참조하면 어떻게 되나요? (렌더링 에러 발생 또는 빈 값 출력. `default` 함수로 방어 필요.)
3.  **Q3.** `helm template` 명령어와 `helm install --dry-run`의 차이는? (`template`은 로컬 렌더링만, `dry-run`은 쿠버네티스 API 서버와 통신하여 유효성 검증까지 수행)

---

다음 주, 템플릿 문법의 끝판왕인 **제어문(If, Range)**과 **Named Template**을 배웁니다.
이제 단순한 치환이 아니라 프로그래밍을 하게 됩니다.

**Templating is Art.**
