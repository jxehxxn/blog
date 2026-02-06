---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 7 - Periodic Tasks (Celery Beat)"
---

Cron은 훌륭하지만, 분산 환경에서는 관리하기 힘듭니다.
서버가 5대인데 5대에서 다 Cron이 돌면? 이메일이 5번 갑니다.

**Celery Beat**는 중앙화된 스케줄러입니다.
"지금 이거 실행할 시간이야"라고 Broker에 메시지만 딱 던져줍니다. 실행은 워커 중 한 놈이 알아서 합니다.

---

## 1. Celery Beat Architecture

Beat는 Worker 안에 포함된 게 아닙니다. (개발 땐 `-B` 옵션으로 같이 띄우지만)
프로덕션에서는 **별도의 프로세스**로 띄워야 합니다.

*   **Beat:** 스케줄을 보고 있다가 시간이 되면 Task를 Broker에 Publish.
*   **Worker:** Broker에서 Task를 Subscribe해서 실행.

**주의:** Beat 프로세스는 **반드시 1개**만 떠 있어야 합니다. 2개 뜨면 작업이 중복 실행됩니다.

---

## 2. Configuration (`beat_schedule`)

`celery.py`나 설정 파일에 정의합니다.

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'send-daily-report': {
        'task': 'tasks.send_report',
        'schedule': crontab(hour=7, minute=30), # 매일 아침 7:30
        'args': (16,),
    },
    'check-health-every-10s': {
        'task': 'tasks.health_check',
        'schedule': 10.0, # 10초마다
    },
}
```

---

## 3. Dynamic Scheduling (Database Backend)

코드로 스케줄을 박아넣으면(Hard-coding), 시간을 바꿀 때마다 배포해야 합니다.
**Django Celery Beat** 확장을 쓰면 DB에 스케줄을 저장하고, 어드민 페이지에서 런타임에 바꿀 수 있습니다.

```bash
pip install django-celery-beat
```

`settings.py`에 `django_celery_beat` 추가하고 마이그레이션하면 끝.

---

## 4. Timezone Issue

서버는 UTC인데 나는 한국 시간(KST)으로 9시에 보내고 싶다면?
Celery는 Timezone 설정에 민감합니다.

```python
app.conf.timezone = 'Asia/Seoul'
app.conf.enable_utc = False # (Django를 쓴다면 USE_TZ 설정과 맞춰야 함)
```

---

## 🛠️ Lab: The Alarm Clock

1.  매분마다 현재 시간을 파일에 기록하는 Task를 만듭니다.
2.  `beat_schedule`에 `crontab(minute='*')`으로 등록합니다.
3.  Beat 프로세스를 띄웁니다: `celery -A myproj beat`
4.  Worker 프로세스를 띄웁니다: `celery -A myproj worker`
5.  1분마다 로그가 찍히는지 확인합니다.

---

## 📝 7주차 과제: Newsletter Scheduler

**목표:** 매주 월요일 오전 9시에 뉴스레터를 발송하는 스케줄러를 구성하세요.

1.  `send_newsletter` Task 작성.
2.  Cron 표현식: `0 9 * * 1` (월요일 09:00).
3.  **심화:** 만약 월요일이 공휴일이면 화요일에 보내고 싶다면?
    *   Cron으로는 불가능합니다.
    *   Task 내부에서 `if is_holiday(): retry(countdown=24*3600)` 로직을 구현해야 합니다. 이 로직을 작성해보세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 프로덕션 환경에서 Celery Beat 프로세스를 2개 이상 실행하면 어떤 문제가 발생하나요? (동일한 주기적 작업이 중복 스케줄링되어 중복 실행됨)
2.  **Q2.** 스케줄 정보를 소스 코드가 아닌 데이터베이스에 저장하여 동적으로 관리하기 위해 사용하는 라이브러리는? (django-celery-beat)
3.  **Q3.** 매월 1일 자정에 실행되는 Crontab 설정은? (`crontab(day_of_month=1, hour=0, minute=0)`)

---

이것으로 Part 2가 끝났습니다.
이제 우리 시스템은 꽤 복잡해졌습니다. 워커가 잘 돌고 있는지, 큐에 작업이 얼마나 쌓였는지 어떻게 알까요?
다음 주, Part 3의 시작은 **모니터링(Flower)**입니다.

**Timely execution.**
