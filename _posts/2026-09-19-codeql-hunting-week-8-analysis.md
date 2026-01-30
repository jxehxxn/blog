---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 8 - Needle in a Haystack (Analyzing Results at Scale)"
---

100개의 레포를 스캔했더니 `results` 폴더에 SARIF 파일이 가득합니다.
열어보니 경고가 5,000개입니다. "와! 취약점 5천개 찾았다!"라고 좋아하면 안 됩니다.
그중 4,950개는 **False Positive (오탐)**이거나 **Trivial (쓸모없음)**일 확률이 높습니다.

이제 우리는 **Data Analyst**가 되어야 합니다.

---

## 📄 SARIF (Static Analysis Results Interchange Format)

SARIF는 정적 분석 결과를 담는 표준 JSON 포맷입니다.
구조가 좀 복잡하지만, 파이썬으로 파싱하면 쉽습니다.

*   `runs[].results[]`: 발견된 취약점 목록
*   `ruleId`: 취약점 종류 (예: `java/sql-injection`)
*   `locations[]`: 파일 경로 및 라인 번호

---

## 🧹 False Positive (FP) Filtering Strategy

쓰레기를 걸러내는 전략입니다.

1.  **Test Code Exclusion:** `/test/`, `/mock/`, `/sample/` 경로에서 발견된 건 무시합니다. (테스트 코드는 원래 취약하게 짭니다.)
2.  **Vendor Code Exclusion:** `node_modules`, `vendor` 등 외부 라이브러리 폴더는 뺍니다. (우리는 메인 프로젝트를 타겟팅합니다.)
3.  **Score Filtering:** CodeQL은 결과마다 신뢰도 점수를 매깁니다. 점수가 낮은 건 일단 미룹니다.
4.  **Heuristics (경험적 필터링):**
    *   Source와 Sink가 같은 함수 안에 있다? -> 로컬 변수일 확률 높음. (원격 공격 불가) -> 제외.
    *   Sink의 인자가 상수 문자열이다? -> 절대 안전함. -> 제외.

---

## 🛠️ Lab: The Sieve (체 거르기)

`sarif_filter.py` 스크립트를 작성합니다.

```python
import json
import glob

sarif_files = glob.glob("results/*.sarif")
total_vulns = 0

for sarif_file in sarif_files:
    with open(sarif_file) as f:
        data = json.load(f)
        
    for run in data.get('runs', []):
        for result in run.get('results', []):
            path = result['locations'][0]['physicalLocation']['artifactLocation']['uri']
            rule_id = result['ruleId']
            
            # 필터링 로직
            if "test" in path or "vendor" in path:
                continue
                
            print(f"[+] Found {rule_id} in {path}")
            total_vulns += 1

print(f"Total actionable vulnerabilities: {total_vulns}")
```
이걸 돌리면 5,000개가 50개로 줄어드는 기적을 볼 수 있습니다. 이제 50개만 보면 됩니다.

---

## 📝 8주차 과제: HTML 리포트 생성

필터링된 결과를 보기 좋게 **HTML 테이블**로 변환하는 기능을 추가하세요.
Repo 이름, 취약점 종류, 파일 경로, 그리고 **GitHub 링크**까지 생성되면 클릭 한 번으로 코드를 확인할 수 있겠죠?

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 정적 분석 도구의 결과 교환을 위한 표준 JSON 포맷의 약어는?
2.  **Q2.** 대량의 분석 결과에서 테스트 코드나 외부 라이브러리 등 관심 없는 결과를 제외하는 과정을 무엇이라 하나요? (F... P... Filtering)
3.  **Q3.** Source와 Sink가 동일한 함수 내에 존재하는 경우, 왜 공격 가능성이 낮다고 판단하나요? (외부 입력 도달 가능성 낮음)

---

다음 주는 **Reporting**입니다.
걸러낸 50개를 직접 눈으로 확인(Triage)하고, 진짜 취약점이라면 **PoC(개념 증명)**를 만들어서 보고서를 써야 합니다.
여기서부터는 자동화가 아니라 **인간의 통찰력**이 필요합니다.

**Filter the noise.**
