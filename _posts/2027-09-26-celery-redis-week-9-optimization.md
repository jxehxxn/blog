---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 9 - Production Best Practices (Worker Tuning, Prefetch)"
---

기본 설정(`celery worker`)만으로 운영하면 반드시 성능 이슈를 만납니다.
작은 작업 100만 개가 있는데 워커가 멍 때리거나, 큰 작업 하나 때문에 다른 작업들이 굶어 죽는 현상.
오늘은 **Prefetch Limit**과 **Concurrency** 튜닝을 통해 처리량을 10배 늘리는 법을 배웁니다.

---

## 1. The Prefetch Limit (`prefetch_multiplier`)

Celery 성능 튜닝의 **알파이자 오메가**입니다.

### 동작 원리
워커는 브로커에서 작업을 가져올 때 하나씩 가져오지 않습니다. 효율성을 위해 **미리 여러 개(Prefetch)**를 가져와서 메모리에 쌓아둡니다.
기본값은 `4`입니다. (Concurrency * 4 만큼 가져옴)

### 문제점
*   **긴 작업(Long tasks):** 워커 A가 1시간 걸리는 작업 1개와, 1초 걸리는 작업 3개를 미리 가져갔습니다.
    *   워커 B는 놀고 있습니다.
    *   워커 A의 1시간 작업이 끝날 때까지 뒤에 있는 3개 작업은 시작도 못 합니다. (**Starvation**)
*   **짧은 작업(Short tasks):** 미리 왕창 가져오는 게 네트워크 오버헤드를 줄여서 좋습니다.

### 최적화 전략
*   **긴 작업 위주:** `task_acks_late=True` + `worker_prefetch_multiplier=1`. (하나씩만 가져와라. 미리 쟁여두지 마라.)
*   **짧은 작업 위주:** 기본값(4) 유지 또는 더 늘림.

---

## 2. Concurrency: Process vs Thread vs Gevent

*   **Prefork (Default):** 프로세스 기반. CPU Bound 작업에 적합. 메모리 많이 먹음.
*   **Eventlet / Gevent:** 그린 스레드(Green Thread). I/O Bound 작업(크롤링, API 호출)에 적합. 프로세스 하나로 수천 개 동시 처리 가능.

**API 요청이 많은 워커라면:**
`celery worker -P gevent -c 1000` (동시성 1000개!)

---

## 3. Memory Leak Prevention

파이썬은 메모리 관리가 완벽하지 않습니다. 워커가 오래 돌면 메모리가 조금씩 셉니다.
일정 횟수 이상 작업을 처리하면 워커 프로세스를 죽이고 새로 띄우는 설정을 씁니다.

```python
worker_max_tasks_per_child = 5000 # 5000개 처리하면 재시작
worker_max_memory_per_child = 400000 # 400MB 넘으면 재시작
```

---

## 🛠️ Lab: Prefetch Effect

Prefetch 설정에 따른 동작 차이를 눈으로 확인합니다.

1.  **시나리오:** 10초 걸리는 작업(`long`) 2개, 1초 걸리는 작업(`short`) 10개를 큐에 넣습니다.
2.  **Case A (Default):** 워커 1개 실행. `short` 작업들이 언제 시작되는지 봅니다. (아마 `long` 뒤에 묶여서 늦게 시작될 겁니다)
3.  **Case B (Prefetch=1):** `celery worker -O fair --prefetch-multiplier=1` 옵션으로 실행.
4.  워커 2개를 띄웠을 때, Case B가 Case A보다 작업 분배가 훨씬 고르게 되는지 확인합니다.

---

## 📝 9주차 과제: Optimization Plan

**목표:** 다음 상황에 맞는 최적의 Celery 설정을 제안하세요.

1.  **상황 A:** 비디오 인코딩 서비스. CPU를 100% 쓰며 작업당 10분 소요.
    *   Concurrency 방식?
    *   Prefetch 설정?
2.  **상황 B:** 웹 크롤러. 외부 사이트 응답을 기다리는 게 대부분이며 작업당 0.5초 소요.
    *   Concurrency 방식?
    *   Prefetch 설정?

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 긴 작업(Long running task)이 큐에 섞여 있을 때, 특정 워커가 작업을 독점하여 다른 워커가 놀게 되는 현상을 막기 위한 `prefetch_multiplier` 값은? (1)
2.  **Q2.** I/O Bound 작업이 대다수인 경우, 동시성 처리를 극대화하기 위해 Prefork 대신 사용할 수 있는 Pool 구현체는? (Gevent 또는 Eventlet)
3.  **Q3.** 워커 프로세스의 메모리 누수(Memory Leak)를 방지하기 위해 주기적으로 프로세스를 재시작하게 하는 설정은? (worker_max_tasks_per_child)

---

다음 주, 중요한 작업과 안 중요한 작업을 섞어두면 안 됩니다.
VIP 고객의 이메일은 즉시 보내고, 일반 로그는 천천히 처리하는 **Routing** 기술을 배웁니다.

**Tune it up.**
