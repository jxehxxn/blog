---
layout: post
title: "GoCD 보충 4: Jenkins → GoCD 마이그레이션"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd migration supplement
---

기존 Jenkins 환경의 GoCD 점진 이주.

## 1. 사전 인벤토리

- Job 수 / 종류 분류.
- Plugin 사용 목록.
- 사용 credentials.
- 평균 빌드 시간.

## 2. 6개월 단계 plan

### Month 1: 설치 + 파일럿
- GoCD HA 설치.
- 사내 표준 plugin 5종.
- 1~3 job 이주.

### Month 2: 표준화
- yaml template.
- Vault 통합.
- Slack 통합.
- 5~10 job 이주.

### Month 3: Scale
- Elastic Agent K8s.
- 30+ job 이주.
- Jenkins 운영 줄임.

### Month 4~5: 본격
- 100+ job 이주.
- Jenkins read-only.

### Month 6: Sunset
- Jenkins 종료.
- 사후 retrospective.

## 3. Job 매핑

| Jenkins | GoCD |
|---------|------|
| Job | Pipeline |
| Build | Stage |
| Step | Task |
| Pipeline (Jenkinsfile) | yaml config |
| Credentials | Secret plugin |
| Slave | Elastic Agent |
| Trigger SCM | Material polling/webhook |
| Notification | Slack plugin |

## 4. 흔한 함정

1. Plugin 대체 누락 (Jenkins-only).
2. 사용자 저항.
3. 너무 빠른 sunset.
4. Multi-branch 변환 복잡.
5. Audit log 형식 다름.

## 5. Quick Win

- VSM 시각화.
- Manual approval 정착.
- Cost 절감 (Elastic Agent).

## 6. 결론

마이그레이션은 도구 변경이 아니라 조직 변경. 6개월, 점진, 사용자 교육 필수.
