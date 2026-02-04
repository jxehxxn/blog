---
layout: post
title: "Kubernetes Package Management with Helm: Week 4 - Advanced Template Logic & Flow Control"
---

Week 3에서 변수 치환(`{{ .Values.key }}`)을 배웠습니다.
하지만 현실은 복잡합니다.
"Ingress는 켜져 있을 때만 생성하고 싶어."
"환경 변수 목록을 리스트로 받아서 루프를 돌리고 싶어."

이때 필요한 것이 **Go Template Language**의 제어문(`if`, `range`)과 스코프(`with`)입니다. 여기서부터가 진짜 Helm 개발입니다.

---

## 1. Flow Control: `if-else`

조건부 리소스 생성의 핵심입니다.

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
{{- end }}
```

**주의할 점 (Whitespace Handling):**
`{{-` 처럼 대시(`-`)를 붙이면, 해당 블록이 차지하는 **공백과 개행 문자(Newline)를 제거**합니다.
YAML은 들여쓰기(Indent)에 민감하므로, 렌더링된 결과물에 빈 줄이 생기거나 들여쓰기가 깨지는 걸 막으려면 이 대시를 잘 써야 합니다.

---

## 2. Looping: `range`

반복문입니다. 환경 변수, ConfigMap 데이터, Secrets 등을 동적으로 생성할 때 씁니다.

### List 순회
```yaml
# values.yaml
env:
  - name: DB_HOST
    value: localhost
  - name: DB_PORT
    value: "5432"

# templates/deployment.yaml
env:
  {{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}
```

### Map(Dictionary) 순회
```yaml
# values.yaml
labels:
  app: web
  tier: frontend

# templates/service.yaml
metadata:
  labels:
    {{- range $key, $val := .Values.labels }}
    {{ $key }}: {{ $val | quote }}
    {{- end }}
```

---

## 3. Scoping: `with`

`with`를 쓰면 점(`.`)이 가리키는 **현재 스코프(Context)**가 바뀝니다. 코드를 짧게 줄여줍니다.

```yaml
{{- with .Values.image }}
image: {{ .repository }}:{{ .tag }}  # .Values.image.repository 라고 안 써도 됨!
{{- end }}
```

**함정:** 스코프 안에서는 상위 객체(`.Release`, `.Chart`)에 접근할 수 없습니다.
해결책: `$` 기호(Root Scope)를 씁니다. `{{ $.Release.Name }}`

---

## 4. Named Templates (`_helpers.tpl`)

중복 코드를 제거(DRY - Don't Repeat Yourself)하는 함수 정의 기능입니다.
주로 `templates/_helpers.tpl` 파일에 정의합니다.

```yaml
# _helpers.tpl
{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}

# deployment.yaml
metadata:
  name: {{ include "mychart.fullname" . }}
```

`include` 함수를 통해 정의된 템플릿을 가져와서 렌더링합니다.

---

## 🛠️ Lab: Conditional ConfigMap

### 시나리오
`values.yaml`에 `configFile`이라는 항목이 있을 때만 ConfigMap을 생성하고, 없으면 생성하지 않는 차트를 만듭니다.

1.  **values.yaml:**
    ```yaml
    createConfig: true
    configData:
      setting1: "on"
      setting2: "off"
    ```
2.  **templates/configmap.yaml:**
    ```yaml
    {{- if .Values.createConfig }}
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ include "mychart.fullname" . }}-config
    data:
      {{- range $k, $v := .Values.configData }}
      {{ $k }}: {{ $v | quote }}
      {{- end }}
    {{- end }}
    ```
3.  `helm template`으로 `createConfig: false`일 때 파일이 아예 안 나오는지 확인하세요.

---

## 📝 4주차 과제: The Helper Function

**목표:** `_helpers.tpl`을 활용하여 모든 리소스에 표준 라벨을 붙이세요.

1.  `_helpers.tpl` 파일에 `mychart.labels`라는 이름의 템플릿을 정의합니다.
    *   `app.kubernetes.io/name`: 차트 이름
    *   `app.kubernetes.io/instance`: 릴리스 이름
    *   `app.kubernetes.io/managed-by`: "Helm"
2.  Deployment, Service, Ingress 파일의 `metadata.labels` 섹션에서 `{{ include "mychart.labels" . }}`를 호출하여 이 라벨들을 적용하세요.
3.  렌더링된 YAML에 라벨 3개가 예쁘게 붙어있는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** `{{- if ... }}`에서 대시(`-`)의 역할은 무엇인가요? (Whitespace/Newline 제거)
2.  **Q2.** `range`나 `with` 블록 내부에서 최상위 루트 객체(`values.yaml`의 뿌리)에 접근하려면 어떤 기호를 써야 하나요? (`$`)
3.  **Q3.** `template` 지시어 대신 `include` 함수를 사용하는 주된 이유는 무엇인가요? (`include`는 출력을 다시 파이프라인(`| indent 4`)으로 넘길 수 있어서 들여쓰기 처리에 유리함)

---

다음 주, 차트 하나로는 부족합니다. DB 차트, Redis 차트를 내 차트 안에 포함시키는 **Subcharts**와 **Dependency Management**를 배웁니다.

**Logic in YAML.**
