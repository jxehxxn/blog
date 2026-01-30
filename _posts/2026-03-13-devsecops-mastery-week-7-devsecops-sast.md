---
layout: post
title: "DevSecOps Mastery: 7주차 - DevSecOps의 시작과 SAST (정적 분석)"
---

드디어 **DevSecOps**의 'Sec'이 등장했습니다.
지금까지 우리는 '어떻게 잘 만들까(Dev)'와 '어떻게 잘 배포할까(Ops)'에 집중했습니다.
이제는 **'어떻게 안전하게 만들까(Sec)'**를 고민할 때입니다.

---

## 🛡️ Enter DevSecOps

보안은 더 이상 최종 관문(Gate)이 아닙니다. 품질(Quality)의 일부입니다.
버그가 있으면 배포를 안 하듯, 보안 취약점이 있으면 배포를 안 해야 합니다.

**Shift Left Security**를 기억하시나요? (1주차 내용)
오늘 우리는 가장 왼쪽, 즉 **소스 코드 단계**에서 보안을 잡는 기술인 **SAST**를 배웁니다.

---

## 🔍 What is SAST?

**SAST (Static Application Security Testing)**는 '화이트박스 테스팅'입니다.
애플리케이션을 실행하지 않고, 소스 코드 자체를 분석하여 취약점을 찾습니다.

*   **무엇을 찾나?**
    *   SQL Injection 패턴
    *   하드코딩된 비밀번호/API 키
    *   버퍼 오버플로우 가능성
    *   취약한 암호화 알고리즘 사용

---

## 🛠️ Tool Spotlight: Semgrep (Semantic Grep)

업계에는 SonarQube, Fortify, Checkmarx 등 훌륭한 도구가 많습니다.
하지만 우리는 **Semgrep**을 사용합니다. 왜냐고요?
*   가볍고 빠릅니다. (CI에 딱입니다)
*   오픈소스이며 무료로 쓸 수 있습니다.
*   설정이 간편합니다. (Docker 한 줄이면 끝)

---

## 💣 실습: Catching Vulnerabilities

일부러 취약한 코드를 만들고, Semgrep으로 잡아봅시다.

### 1. 취약한 코드 작성 (`vulnerable.py`)
프로젝트에 다음 파이썬 파일을 추가합니다.

```python
# vulnerable.py
def connect_db():
    # 헉! 비밀번호가 하드코딩 되어 있네요!
    password = "super_secret_password_123" 
    print(f"Connecting with {password}")

def get_user(user_id):
    # 헉! SQL Injection 취약점이 있네요!
    query = "SELECT * FROM users WHERE id = " + user_id 
    execute(query)
```

### 2. Jenkinsfile에 SAST 스테이지 추가

```groovy
stage('Security Scan (SAST)') {
    steps {
        // 도커를 이용해 Semgrep 실행
        // --error 옵션: 취약점이 발견되면 종료 코드 1을 리턴 (빌드 실패)
        sh 'docker run --rm -v $(pwd):/src returntocorp/semgrep semgrep scan --config=p/security-audit --error'
    }
}
```

### 3. 결과 확인
빌드를 돌려보세요. **실패(Failure)**해야 정상입니다.
로그를 보면 Semgrep이 "Hardcoded password"와 "SQL Injection"을 정확히 지적할 것입니다.

---

## 📝 7주차 과제: 취약점 수정하기 (Fix it!)

빌드가 실패한 채로 둘 순 없죠. 코드를 수정해서 파이프라인을 초록불로 만드세요.

1.  비밀번호를 환경 변수(`os.getenv('DB_PASSWORD')`)에서 가져오도록 수정하세요.
2.  수정 후 다시 커밋&푸시하여 파이프라인이 통과하는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 애플리케이션을 실행하지 않고 소스 코드 상태에서 취약점을 분석하는 방식을 약어로 무엇이라 하나요?
2.  **Q2.** 우리가 실습에서 사용한 가볍고 빠른 오픈소스 SAST 도구의 이름은 무엇인가요?
3.  **Q3.** 소스 코드에 비밀번호나 API 키를 직접 적어놓는 위험한 코딩 관습을 무엇이라 하나요? (Hard-...)

---

다음 주는 **컨테이너 보안**과 **DAST(동적 분석)**입니다.
우리가 만든 도커 이미지 자체에 구멍은 없는지, 그리고 실행 중인 애플리케이션을 해커처럼 찔러보는(Attacking) 방법을 배웁니다.

**Secure Code, Secure Life.**
