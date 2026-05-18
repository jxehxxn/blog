---
layout: post
title: "GoCD 심화 Week 7: Plugin 개발 기초 (Java)"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd plugin senior
---

## 학습 목표
- Plugin 아키텍처.
- Plugin SDK + 첫 plugin.
- 빌드 + 설치.

## 1. 아키텍처
- Plugin = jar file.
- `plugins/external/` 에 두면 자동 load.
- 서버 API: REST + lifecycle hook.

## 2. Plugin Types
- Authentication
- Authorization
- SCM
- Task
- Notification
- Elastic Agent
- Secret
- Artifact
- Configuration (YAML/JSON/Groovy)

각 type별 interface.

## 3. SDK

ThoughtWorks 공식 GitHub: `gocd/go-plugin-api-extensions`.

```xml
<dependency>
  <groupId>cd.go.plugin</groupId>
  <artifactId>go-plugin-api</artifactId>
  <version>...</version>
</dependency>
```

## 4. Hello Plugin

```java
@Extension
public class HelloPlugin implements GoPlugin {
    public void initializeGoApplicationAccessor(GoApplicationAccessor accessor) {}

    public GoPluginIdentifier pluginIdentifier() {
        return new GoPluginIdentifier("notification", List.of("1.0"));
    }

    public GoPluginApiResponse handle(GoPluginApiRequest request) {
        // request.requestName 기반 분기
        return new DefaultGoPluginApiResponse(200, "{\"status\":\"ok\"}");
    }
}
```

## 5. Build + Install

```bash
mvn package
cp target/hello-plugin.jar /var/lib/go-server/plugins/external/
systemctl restart go-server
```

UI Admin → Plugins에서 등장 확인.

## 6. plugin.xml

```xml
<go-plugin id="cd.go.contrib.hello" version="1">
  <about>
    <name>Hello Plugin</name>
    <version>1.0.0</version>
  </about>
</go-plugin>
```

## 7. Logging

`/var/log/go-server/plugins/cd.go.contrib.hello.log`.

## 8. 자가평가
### Q1. Plugin = ? **jar file**. 정답 1.
### Q2. 위치? **plugins/external/**. 정답 1.
### Q3. Plugin types 수? **9+종**. 정답 1.
### Q4. interface? **GoPlugin**. 정답 1.
### Q5. plugin.xml 필수? **id + version**. 정답 1.
