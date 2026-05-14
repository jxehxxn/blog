---
layout: post
title: "TruffleHog Week 4: Git 히스토리 깊게 파기 — pack file, dangling commit, BFG와의 관계"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- Git이 객체를 저장하는 방식(blob, tree, commit, pack)을 안다.
- `git rm`이 왜 시크릿을 못 지우는지 직접 실험한다.
- TruffleHog로 전체 히스토리를 스캔하는 옵션을 정확히 사용한다.
- BFG Repo-Cleaner, `git filter-repo`, `git filter-branch`의 차이와 적용 시점을 안다.

## 1. Git의 진실 — 모든 객체는 영원하다 (until GC)

비유: Git 리포는 **나무에 묶인 종이쪽지 한 무더기**입니다. 종이를 뜯는다고 나무에서 사라지는 게 아니라, 그냥 가지에 연결된 종이가 줄어들 뿐입니다. 종이 자체는 아래에 떨어져 쌓여 있고, 나중에 누군가 줍거나(`git reflog`) 청소(GC)할 때까지 머물러 있습니다.

기술적으로:

- 모든 파일 내용은 **blob**으로 저장 (SHA1 키)
- 디렉토리는 **tree**로 저장
- 커밋은 **commit** 객체
- 위 3종이 모여 **pack file**로 압축 보관 (`.git/objects/pack/`)

`git rm secret.txt` 후 `git commit`을 해도 이전 커밋의 blob은 그대로 살아 있습니다. **그 commit에 도달 가능한 한 영원히** 살아 있고, brunch가 모두 옮겨가도 reflog에 90일은 남아 있습니다.

## 2. 직접 실험 — `git rm`이 못 지운다는 증거

```bash
mkdir secret-test && cd secret-test
git init
echo "AKIAYVP4CIPPERUVIFXG" > key.txt
git add key.txt && git commit -m "add key"

# 지우기 시도
git rm key.txt
git commit -m "remove key"

# blob은 살아 있는가?
git cat-file -p HEAD^:key.txt
# AKIAYVP4CIPPERUVIFXG  ← 출력됨!
```

TruffleHog로 확인:

```bash
trufflehog git file://$(pwd) --only-verified=false --json | jq '.Raw'
# "AKIAYVP4CIPPERUVIFXG"
```

**시크릿은 여전히 존재합니다.** 빅테크에서 가장 흔한 사고 시나리오: "지웠다고 보고했는데 GitGuardian 알람이 안 꺼져요" — 위와 같은 이유입니다.

## 3. TruffleHog의 git 모드 — 옵션 한 줄도 빠짐없이

```bash
trufflehog git <repo> \
  --since-commit=<sha>   # 특정 커밋 이후만
  --branch=<name>        # 특정 브랜치만
  --max-depth=<N>        # 최대 N개 커밋만
  --no-update            # 디텍터 자동 업데이트 끔
  --concurrency=16       # 동시 스캔 워커 수
  --filter-unverified    # 같은 시크릿이 여러 번 나오면 unverified 중복 제거
  --bare                 # bare repo도 스캔
```

빅테크 패턴:
- **PR CI**: `--since-commit=$(git merge-base origin/main HEAD)` 로 변경분만 스캔 → 빠르고 결정적.
- **주간 풀스캔**: 별도 cron으로 모든 브랜치 + reflog 까지 풀스캔 → 누락 캐치.

## 4. Dangling commit & reflog까지 — 진짜 풀스캔

```bash
# 모든 ref 강제 포함
git fsck --unreachable --no-reflog
# dangling commit 목록

# reflog까지 포함해 풀 객체 스캔
trufflehog git file://$(pwd) --bare
```

TruffleHog는 `--bare` 모드에서 reachable 객체 위주로 도는데, 완전한 누락 방지를 위해선 다음을 함께 합니다.

```bash
git for-each-ref --format='%(refname)' > all-refs.txt
git pack-refs --all
git reflog expire --expire=now --all
```

(주의: 위 명령은 reflog를 비웁니다. 운영 리포에선 신중히.)

## 5. 이미 누출된 시크릿 — 지우는 4가지 도구

### (a) `git filter-branch` (레거시)

```bash
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch key.txt" \
  --prune-empty --tag-name-filter cat -- --all
```

느리고, 위험하고, Git 공식 문서가 "사용하지 말라"고 권고. 학습용으로만.

### (b) BFG Repo-Cleaner (실무 표준)

```bash
java -jar bfg.jar --delete-files key.txt my-repo.git
git -C my-repo.git reflog expire --expire=now --all
git -C my-repo.git gc --prune=now --aggressive
```

특정 파일 삭제, 문자열 치환, 큰 파일 제거에 최적. 빅테크 인시던트 대응에서 가장 많이 쓰입니다.

### (c) `git filter-repo` (현대 권장)

```bash
git filter-repo --invert-paths --path key.txt
```

filter-branch 후속. Git 공식 권장. BFG보다 더 정교한 변형 지원.

### (d) GitHub의 "Secret scanning" 알람 + 토큰 회수

도구로 히스토리를 다듬어도 **이미 노출된 토큰은 살아 있다는 가정으로 즉시 회수(revoke)**해야 합니다. 10주차 인시던트 대응에서 다룹니다.

## 6. 빅테크 현장 사례 — Toyota 2022 재해석

Toyota는 GitHub에 5년 전 커밋된 액세스 토큰이 노출되어 있었습니다. 흥미로운 점:

1. 토큰을 추가한 커밋은 5년 전.
2. 그 사이 수많은 커밋이 쌓였고, 표면적으로는 `git log -p`로 찾기 어려움.
3. 누군가 fork·mirror한 곳에 그대로 남음 — Toyota 본 리포만 정리해선 안 됨.

TruffleHog `github` 소스 모드는 **fork와 forked-from**을 모두 따라가는 옵션이 있습니다.

```bash
trufflehog github \
  --org=mycorp \
  --include-forks \
  --include-paths='.*' \
  --concurrency=32 \
  --only-verified
```

6주차에서 org 단위 운영을 다룹니다.

## 7. 실습 과제

1. 본인 GitHub에 테스트 리포를 만들고, 1번 커밋에 가짜 AWS 키를 넣고, 2번 커밋에서 `git rm`으로 지움.
2. `trufflehog git file://...`로 detection 결과를 확인.
3. BFG 또는 `git filter-repo`로 히스토리에서 제거.
4. 같은 명령을 재실행해서 detection이 사라졌는지 검증.
5. 만약 같은 리포를 동료가 fork했다면 어떻게 알릴 것인가? 한 단락으로 응답 계획 작성.

## 8. 자가평가 퀴즈

### Q1. `git rm` 후 commit하면 이전 blob은?
1. 즉시 디스크에서 삭제된다.
2. unreachable 상태로 남아 있다가 GC 시점에 삭제된다.
3. 절대 삭제되지 않는다.
4. force push 시점에 자동 삭제.

**정답:** 2번. reachable한 동안엔 살아 있고, unreachable이 되어도 `gc.pruneExpire`(기본 2주) 이후에 GC됩니다.

### Q2. `--since-commit=<merge-base>`를 PR CI에서 쓰는 이유는?
1. 결정성 + 변경분만 스캔.
2. 비용 절감.
3. 같은 시크릿 중복 보고 방지.
4. 모두 맞음.

**정답:** 4번.

### Q3. BFG와 `git filter-repo`의 차이는?
1. BFG는 Java, filter-repo는 Python.
2. filter-repo는 더 정교한 변형 가능, Git 공식 권장.
3. BFG는 큰 파일 삭제에 특화.
4. 모두 맞음.

**정답:** 4번.

### Q4. fork된 리포에 있는 시크릿은?
1. 본 리포에서 지우면 자동으로 사라진다.
2. 본 리포 정리와 무관하게 별도 처리 필요.
3. GitHub이 자동으로 처리.
4. TruffleHog가 자동 회수.

**정답:** 2번. fork는 독립 리포입니다.

### Q5. 히스토리 다듬기보다 더 우선되어야 하는 일은?
1. force push.
2. 누출된 자격증명의 즉시 회수(revoke).
3. PR 리뷰어 공개 비난.
4. README 업데이트.

**정답:** 2번. 히스토리를 아무리 깨끗이 해도 이미 누군가 클론·복사했다면 그 시크릿은 살아 있습니다. 가장 먼저 권한을 죽이세요.

## 9. 다음 주차

[Week 5: CI/CD 통합]에서는 pre-commit/GitHub Actions/GitLab CI/Jenkins에 TruffleHog를 안전하게 박는 패턴을 다룹니다. "PR 차단" vs "리포팅 전용"의 선택 기준도 함께.
