---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 6 - Mass Production (Building the Auto-Scan Pipeline)"
---

손으로 `git clone`하고, `codeql database create`를 치는 건 하수나 하는 짓입니다.
우리에겐 100개의 타겟이 있습니다. 자고 일어났을 때 100개의 분석 결과가 나와 있어야 합니다.
이것이 **"Mass Scanning Pipeline"**입니다.

---

## 🏭 The Pipeline Architecture

파이프라인은 크게 3단계로 나뉩니다.

1.  **Ingestion:** `targets.txt`를 읽어서 `git clone`.
2.  **Build & Extract:** `codeql database create` 명령어로 DB 생성.
    *   **난관:** Java나 C++은 빌드 명령어(`mvn`, `gradle`, `make`)를 알아내야 합니다.
    *   **해법:** CodeQL CLI의 `--command` 옵션을 비워두면(Auto-build 모드), CodeQL이 알아서 빌드 시스템을 추측합니다. (완벽하진 않지만 쓸만함)
3.  **Analyze:** `codeql database analyze` 명령어로 쿼리 실행 -> SARIF 출력.

---

## 🔧 Handling Build Failures

자동화의 적은 **빌드 실패**입니다.
*   "의존성이 없어요", "자바 버전이 안 맞아요"...
*   모든 걸 고칠 순 없습니다. **"Fail Fast, Skip Next"** 전략을 씁니다.
    *   Auto-build 시도 -> 실패하면 로그 남기고 다음 타겟으로.
    *   우리는 100개 중 30개만 성공해도 충분히 많습니다.

---

## 🛠️ Lab: The Factory (Scripting)

파이썬으로 `auto_scan.py`를 작성합니다.

```python
import os
import subprocess

with open("targets.txt") as f:
    repos = f.read().splitlines()

for repo in repos:
    name = repo.split("/")[-1]
    # 1. Clone
    subprocess.run(["git", "clone", repo])
    
    # 2. Create DB (Auto-build)
    db_path = f"dbs/{name}"
    cmd = f"codeql database create {db_path} --language=java --source-root={name}"
    ret = subprocess.run(cmd, shell=True)
    
    if ret.returncode == 0:
        # 3. Analyze
        subprocess.run(f"codeql database analyze {db_path} java-security-extended.qls --format=sarif-latest --output=results/{name}.sarif", shell=True)
```
이 스크립트를 밤새 돌려놓으세요.

---

## 📝 6주차 과제: 파이프라인 확장

위 스크립트에 **Timeout** 기능을 추가하세요.
어떤 프로젝트는 빌드가 1시간씩 걸릴 수도 있습니다. 10분 이상 걸리면 강제로 죽이고(`kill`) 다음으로 넘어가도록 만드세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** CodeQL CLI에서 빌드 명령어를 명시하지 않고 자동으로 추론하게 하는 기능을 무엇이라 하나요? (A...)
2.  **Q2.** CodeQL 분석 결과를 저장하는 표준 JSON 포맷 파일의 확장자는? (.s...)
3.  **Q3.** 대규모 스캔 시 빌드 실패가 발생했을 때 권장되는 전략은? (하나하나 고친다 vs 빠르게 건너뛴다)

---

다음 주는 **Custom Queries**입니다.
CodeQL이 기본으로 제공하는 쿼리(`security-extended`)만 돌리면 남들이 다 찾은 것밖에 안 나옵니다.
나만의 0-day를 찾으려면, 나만의 쿼리를 짜야 합니다.

**Scale out.**
