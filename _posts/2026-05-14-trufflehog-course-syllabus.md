---
layout: post
title: "Big Tech에서의 TruffleHog 활용: 12주 강의 계획서"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 강의 개요

이 강의는 글로벌 빅테크(FAANG, MAGMA 급) 환경에서 **TruffleHog**를 운영하는 보안 엔지니어/DevSecOps 엔지니어를 양성하는 12주 과정입니다. 단순히 `trufflehog` 명령을 실행하는 수준이 아니라, **수천 개의 리포지토리·CI 파이프라인·다중 클라우드 환경**에서 시크릿 누출을 탐지·검증·차단·대응하는 전체 라이프사이클을 다룹니다.

### 사전 요구 지식

- Linux 셸 사용 (필수)
- Git 워크플로우 이해 (필수)
- 기본 정규표현식 (권장)
- Go 또는 Python 기초 (커스텀 디텍터 작성 주차)
- CI/CD 개념 (GitHub Actions 또는 GitLab CI)

### 학습 결과

12주를 마치면 수강생은 다음을 할 수 있습니다.

1. 조직 단위(GitHub Organization 1,000+ repo)에서 TruffleHog를 운영 배포
2. 자체 토큰 포맷에 맞춘 커스텀 디텍터 + 라이브 검증기(verifier) 구현
3. False Positive 비율을 5% 이하로 유지하는 튜닝
4. 시크릿 누출 인시던트의 5단계 대응(탐지→격리→회수→로테이션→포스트모템)
5. SOC 2 / PCI-DSS / HIPAA 감사 증빙 자동 생성
6. SIEM(Splunk, Chronicle), Vault, SOAR(Tines)와의 통합 파이프라인 구축

## 주차별 주제

| 주차 | 주제 | 형식 |
|------|------|------|
| 1주 | 오리엔테이션: 왜 시크릿 스캐닝이 빅테크의 0순위 통제인가 | 강의 + 토론 |
| 2주 | TruffleHog 기초: 설치, 첫 스캔, 출력 해석 | 실습 |
| 3주 | 탐지 엔진 내부: detector, verifier, entropy의 삼각관계 | 강의 + 코드 분석 |
| 4주 | Git 히스토리 깊게 파기: pack file, dangling commit, BFG와의 관계 | 실습 |
| 5주 | CI/CD 통합: pre-commit, GitHub Actions, GitLab CI, Jenkins | 실습 |
| 6주 | 조직 규모 스캔: GitHub org, GitLab group, monorepo 전략 | 실습 + 강의 |
| 7주 | 커스텀 디텍터 작성: regex, verifier, 사내 토큰 포맷 | 실습 |
| 8주 | 도구 연동: Splunk SIEM, HashiCorp Vault, Tines SOAR | 실습 |
| 9주 | False Positive 튜닝: allowlist, baseline, confidence 모델 | 강의 + 실습 |
| 10주 | 시크릿 누출 인시던트 대응: NIST 800-61 매핑 | 시뮬레이션 |
| 11주 | 엔터프라이즈/거버넌스: 감사·컴플라이언스·메트릭 | 강의 |
| 12주 | 캡스톤: Big Tech 환경 시뮬레이션 풀스택 구축 | 프로젝트 |

## 평가

- 매 주차 자가평가 퀴즈 (5문항, 정답+해설 제공): 30%
- 주차별 실습 과제 제출: 30%
- 캡스톤 프로젝트: 40%

캡스톤은 3가지 채점 기준(설계 적정성, 운영 완성도, 인시던트 대응 시나리오)으로 평가하며, 3종 모범답안(스타트업/스케일업/엔터프라이즈)이 제공됩니다.

## 보충 포스트 (Supplementary)

본 강의에서 "심화는 보충 자료 참조"라고 언급된 모든 주제는 별도 포스트로 작성되어 있습니다.

- 보충 1: 탐지 엔진 내부 — entropy 기반 vs regex 기반의 정량 비교
- 보충 2: Verifier 설계 — HTTP 호출, idempotency, rate limit, 캐싱
- 보충 3: Pre-receive hook으로 monorepo 차단막 만들기
- 보충 4: 컴플라이언스 매핑 (SOC 2 CC6.1, PCI-DSS 3.5, HIPAA §164.312)
- 보충 5: 캡스톤 채점 루브릭 + 3종 모범답안 상세

## 사용 도구 버전

- TruffleHog v3.x (Go 구현체, 2026년 4월 기준 최신)
- pre-commit 3.x
- GitHub Actions runner ubuntu-22.04
- HashiCorp Vault 1.16
- Splunk Enterprise 9.2 (또는 Splunk Cloud)

## 첫 주차 안내

다음 포스트 [Week 1: 오리엔테이션]에서는 **왜 시크릿 스캐닝이 빅테크에서 "있으면 좋은 것"이 아니라 "없으면 회사가 망하는 것"인지**를 Uber 2016, Toyota 2022, Twitch 2021 실제 사례로 풀어봅니다.
