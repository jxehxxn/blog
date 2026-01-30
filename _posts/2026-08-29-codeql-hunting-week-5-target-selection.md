---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 5 - The Hunter's Eye (Target Selection & Automation)"
---

사냥꾼에게 가장 중요한 건 총(CodeQL)이 아닙니다. **사냥감(Target)**입니다.
아무리 좋은 쿼리를 짜도, 이미 보안 감사가 100번 끝난 리눅스 커널을 찌르면 아무것도 안 나옵니다.
그렇다고 아무도 안 쓰는 토이 프로젝트를 찌르면? CVE를 안 줍니다.

우리는 **"가치가 있으면서도, 아직 털리지 않은"** 황금 타겟을 찾아야 합니다.

---

## 🎯 Selection Criteria (The Golden Ratio)

좋은 타겟의 4조건:
1.  **Popularity (인기도):** 스타 1000개 이상, 다운로드 수 높음. (그래야 파급력이 있음)
2.  **Activity (활동성):** 최근 3개월 내 커밋 있음. (죽은 프로젝트는 제보해도 안 고쳐줌)
3.  **Language (언어):** 내가 잘 아는 언어, 그리고 CodeQL이 지원하는 언어(Java, Go, Py, JS, C++).
4.  **Security Maturity (보안 성숙도):** **적당히 낮아야 함.**
    *   `.github/workflows/codeql.yml` 파일이 이미 있다? -> 이미 CodeQL이 매일 돌아가고 있다는 뜻. 피하세요.
    *   보안 정책(`SECURITY.md`)은 있는데 자동화된 툴은 안 보인다? -> **최고의 타겟.**

---

## 🤖 Automation Tools (사냥감 탐색)

일일이 GitHub 검색창을 뒤질 순 없습니다. API를 씁시다.

### GitHub API Search
```bash
# 예: 최근 업데이트된 자바 웹 프레임워크 검색
curl "https://api.github.com/search/repositories?q=language:java+topic:web-framework+pushed:>2026-01-01&sort=stars"
```

### GHSA (GitHub Security Advisories)
과거의 CVE 기록을 봅니다.
"최근에 A 라이브러리에서 SQL Injection이 터졌네? 비슷한 B 라이브러리도 터지지 않을까?"
이것을 **Variant Analysis(변종 분석)**라고 합니다. 아주 확률 높은 접근법입니다.

---

## 🛠️ Lab: The List Builder

Python과 `PyGithub` 라이브러리를 이용해 타겟 리스트를 자동으로 뽑아봅시다.

```python
from github import Github

g = Github("YOUR_TOKEN")
query = "language:java stars:1000..5000 pushed:>2026-01-01"
repos = g.search_repositories(query)

for repo in repos[:50]:
    # CodeQL 워크플로우가 있는지 체크
    try:
        repo.get_contents(".github/workflows/codeql.yml")
        print(f"[-] Skip {repo.full_name} (Already has CodeQL)")
    except:
        print(f"[+] Target Found: {repo.full_name}")
```
이 스크립트를 돌리면 순식간에 **"공략 가능한 타겟 리스트"**가 `targets.txt`에 쌓입니다.

---

## 📝 5주차 과제: 타겟 100개 확보

여러분이 주력으로 삼을 언어(예: Python)를 정하고, 위 스크립트를 수정하여 타겟 레포지토리 URL 100개를 확보하세요.
이 리스트는 다음 주 파이프라인 실습의 **입력 데이터**가 됩니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 타겟 선정 시, 이미 해당 프로젝트에 CodeQL 워크플로우가 적용되어 있는지 확인하는 이유는 무엇인가요? (중복 방지 / 난이도 조절)
2.  **Q2.** 과거에 발견된 취약점 패턴을 다른 유사한 프로젝트에서 찾아보는 기법을 무엇이라 하나요? (V... A...)
3.  **Q3.** GitHub에서 프로젝트의 보안 이슈(CVE) 정보를 모아놓은 데이터베이스의 이름은? (G...)

---

다음 주는 **Pipeline Construction**입니다.
`targets.txt`에 있는 100개의 레포를 자동으로 다운받고, 빌드하고, CodeQL DB로 굽는 공장을 세웁니다.

**Target acquired.**
