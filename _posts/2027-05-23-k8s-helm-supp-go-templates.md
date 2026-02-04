---
layout: post
title: "Kubernetes Package Management with Helm: Supplement - Go Template Cheat Sheet"
---

Week 4에서 다룬 Go Template 문법은 Helm의 꽃이자 악몽입니다.
매번 공식 문서를 뒤지기 귀찮은 여러분을 위해, 실무에서 가장 자주 쓰는 패턴을 모았습니다.
이 페이지를 즐겨찾기 해두세요.

---

## 1. Variables & Pipelines

```yaml
# 변수 선언 및 할당
{{- $name := .Release.Name -}}

# 파이프라인 (함수 체이닝)
{{ .Values.image.tag | default "latest" | quote }}
{{ .Values.foo | upper | quote }}
```

---

## 2. Flow Control (If-Else)

```yaml
# 기본
{{- if .Values.enabled }}
enabled: true
{{- else }}
enabled: false
{{- end }}

# Not (부정)
{{- if not .Values.enabled }}
disabled: true
{{- end }}

# And / Or
{{- if and .Values.foo .Values.bar }}
both: true
{{- end }}

{{- if or .Values.foo .Values.bar }}
either: true
{{- end }}

# Equality (같은가?)
{{- if eq .Values.env "production" }}
production: true
{{- end }}
```

---

## 3. Loops (Range)

```yaml
# 리스트 반복 (Item만 필요할 때)
{{- range .Values.ingress.hosts }}
- {{ . }}
{{- end }}

# 리스트 반복 (Index도 필요할 때)
{{- range $index, $host := .Values.ingress.hosts }}
{{ $index }}: {{ $host }}
{{- end }}

# 맵(Dictionary) 반복
{{- range $key, $val := .Values.labels }}
{{ $key }}: {{ $val }}
{{- end }}
```

---

## 4. Scoping (With)

```yaml
# 스코프 변경
{{- with .Values.ingress }}
enabled: {{ .enabled }}  # .Values.ingress.enabled 아님!
path: {{ .path }}
release: {{ $.Release.Name }} # 루트 스코프 접근 ($ 필수)
{{- end }}
```

---

## 5. Indentation (들여쓰기 지옥 탈출)

`include`로 가져온 텍스트 블록을 YAML 구조에 맞게 들여쓰기할 때 필수입니다.

```yaml
# _helpers.tpl
{{- define "my.labels" -}}
app: web
tier: frontend
{{- end -}}

# deployment.yaml
metadata:
  labels:
    {{- include "my.labels" . | nindent 4 }}
```
*   `indent 4`: 앞에 공백 4칸 추가.
*   `nindent 4`: **개행(Newline) 후** 공백 4칸 추가. (이걸 제일 많이 씀)

---

## 6. Debugging

렌더링이 자꾸 이상하게 나올 때, 중간중간 값을 찍어보세요.

```yaml
# 렌더링 도중 에러를 내고 멈춤 (값 확인용)
{{- fail "Stop here!" -}}

# 객체를 YAML로 덤프 (구조 확인용)
{{ .Values | toYaml }}
```

이 치트시트만 있으면 웬만한 차트는 다 만들 수 있습니다.
