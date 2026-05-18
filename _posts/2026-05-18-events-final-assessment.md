---
layout: post
title: "Events 최종 자가평가 — 30문항"
date: 2026-05-18 23:00:00 +0900
categories: argo-events assessment
---

각 문항 정답 1.

## Q1. 3 컴포넌트?
**EventBus/EventSource/Sensor**.

## Q2. EventBus 기술?
**NATS JetStream**.

## Q3. EventSource 책임?
**외부 → bus**.

## Q4. Sensor 책임?
**bus 구독 + 조건 평가 + trigger**.

## Q5. webhook secret?
**HMAC**.

## Q6. GitHub auto webhook?
**repo에 자동 등록**.

## Q7. Kafka consumerGroup?
**offset 관리**.

## Q8. S3 trigger?
**ObjectCreated**.

## Q9. cron 대체?
**calendar source**.

## Q10. Multi-dep AND?
**conditions a && b**.

## Q11. parameter src?
**dependency + dataKey JSONPath**.

## Q12. k8s trigger?
**create/update/patch/delete**.

## Q13. argoWorkflow trigger op?
**submit/resume/retry**.

## Q14. http trigger?
**SOAR/Slack 외부**.

## Q15. lambda trigger?
**AWS Lambda invoke**.

## Q16. filter time?
**업무 시간**.

## Q17. NATS auth?
**token/nkey**.

## Q18. Sensor RBAC?
**최소 권한 SA**.

## Q19. EventBus HA?
**3 replica + persistence**.

## Q20. webhook scale?
**HPA**.

## Q21. Kafka scale?
**partition 수**.

## Q22. backpressure?
**JetStream ack**.

## Q23. EventBus 장애?
**전체 자동화 마비**.

## Q24. Jenkins trigger 매핑?
**SCM/webhook/cron → source 종류**.

## Q25. Migration plan?
**parallel 6개월**.

## Q26. 가장 흔한 source?
**webhook (CI)**.

## Q27. Argo Workflow 결합?
**source → sensor → workflow trigger**.

## Q28. Slack trigger?
**channel notification**.

## Q29. idempotent?
**중복 event 처리 X**.

## Q30. 모니터링?
**JetStream metric**.

다음: Multi-cluster fleet.
