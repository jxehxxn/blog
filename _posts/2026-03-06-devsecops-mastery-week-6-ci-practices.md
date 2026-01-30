---
layout: post
title: "DevSecOps Mastery: 6주차 - 지속적 통합(CI) 실무와 베스트 프랙티스"
---

빌드만 된다고 CI가 아닙니다.
**"Continuous Integration (지속적 통합)"**의 진짜 의미는, **"매일, 자주, 코드를 합치고 검증한다"**는 행위 그 자체입니다.
오늘은 진정한 CI를 위한 두 가지 기둥, **테스트 자동화**와 **아티팩트(결과물) 관리**를 배웁니다.

---

## 🧪 Unit Testing Integration (안전망 구축)

테스트 코드가 없는 배포는 브레이크 없는 스포츠카와 같습니다. 빠르지만, 언젠가 처참하게 사고가 납니다.

### JUnit Plugin 활용
Jenkins는 테스트 결과를 시각적으로 보여주는 기능이 뛰어납니다. 보통 Java 진영의 `JUnit` XML 포맷을 표준으로 사용합니다. (Node.js, Python 등 다른 언어도 JUnit 포맷으로 결과를 뱉어내게 할 수 있습니다.)

```groovy
stage('Test') {
    steps {
        // 테스트 실행 (실패하더라도 파이프라인이 바로 멈추지 않게 try-catch를 쓰거나 옵션을 줄 수 있음)
        sh 'npm test -- --reporter junit --output-file test-results.xml'
    }
    post {
        always {
            // 결과 리포트 수집
            junit 'test-results.xml'
        }
    }
}
```
이렇게 설정하면 Jenkins 빌드 화면에 멋진 **Test Result Trend** 그래프가 생깁니다.

---

## 📦 Artifact Management (결과물 관리)

힘들게 빌드한 `jar` 파일이나 `exe` 파일, 어디에 두시나요?
빌드 워크스페이스에 그대로 두면 다음 빌드 때 덮어씌워지거나 사라집니다.

**archiveArtifacts** 명령어를 사용하여 중요 결과물을 보관해야 합니다.

```groovy
stage('Package') {
    steps {
        sh 'npm run build' // dist 폴더에 결과물 생성
    }
    post {
        success {
            // dist 폴더의 모든 파일을 아카이브
            archiveArtifacts artifacts: 'dist/**/*', fingerprint: true
        }
    }
}
```
이제 Jenkins 빌드 페이지 우측 상단에서 언제든지 해당 빌드의 결과물을 다운로드할 수 있게 됩니다.

---

## 🛠️ 실습: The Complete CI Loop

이번 실습은 조금 다이나믹합니다. **고의로 실패**해 볼 겁니다.

1.  여러분의 코드에 일부러 틀린 테스트 케이스를 하나 추가하세요. (예: `expect(1+1).toBe(3)`)
2.  코드를 커밋하고 푸시합니다.
3.  Jenkins가 자동으로 돌면서 테스트 단계에서 **빨간불(실패)**을 띄우는지 확인하세요.
4.  JUnit 리포트에서 어떤 테스트가 깨졌는지 확인하세요.
5.  테스트를 올바르게 수정(`expect(1+1).toBe(2)`)하고 다시 푸시합니다.
6.  **초록불(성공)**이 뜨고, 아티팩트가 생성되는지 확인하세요.

이 사이클을 경험하는 것이 CI의 핵심입니다.

---

## 🔔 Notifications (알림)

빌드가 깨졌는데 아무도 모르면 소용이 없습니다.
이메일, 슬랙(Slack), 팀즈(Teams) 등으로 알림을 보내야 합니다.

```groovy
post {
    failure {
        echo '🚨 Build Failed! Check the logs.'
        // 실제로는: slackSend channel: '#ci-alerts', message: "Build Failed: ${env.BUILD_URL}"
    }
    fixed {
        echo '✅ Build is back to normal!'
    }
}
```
`fixed` 조건은 "이전에는 실패했는데 이번에 성공했을 때" 발동합니다. 기분 좋은 알림이죠.

---

## 📝 6주차 과제: 알림 시스템 구축

(실제 슬랙 연동은 토큰이 필요하므로) `post` 섹션을 활용하여 빌드 상태가 변할 때마다 로그(`echo`)를 다르게 찍어보세요.
*   성공(Success)
*   실패(Failure)
*   복구(Fixed)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Jenkins에서 테스트 결과 리포트를 시각화하기 위해 가장 널리 사용되는 표준 포맷은 무엇인가요? (XML 기반)
2.  **Q2.** 빌드 결과물(바이너리, 패키지 등)을 Jenkins 서버에 영구적으로 보관하기 위해 사용하는 파이프라인 명령어는 무엇인가요?
3.  **Q3.** 이전에 실패했던 빌드가 다시 성공했을 때만 실행되는 `post` 조건(condition)의 이름은 무엇인가요?

---

다음 주, 드디어 과정의 반환점입니다.
그리고 **보안(Security)**이 본격적으로 등장합니다. 소스 코드를 분석해 해커보다 먼저 취약점을 찾아내는 **SAST** 기술을 만날 준비 하세요.

**Build. Test. Archive.**
