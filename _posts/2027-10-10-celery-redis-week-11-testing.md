---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 11 - Testing Celery Tasks"
---

"테스트 코드 돌리는데 Redis를 띄워야 하나요?"
"Task가 비동기로 돌아서 테스트가 안 끝나요."

Celery 테스트의 핵심은 **"비동기를 동기로 만드는 것"**과 **"외부 의존성(Broker)을 끊는 것"**입니다.

---

## 1. Eager Mode (`task_always_eager`)

이 설정을 켜면 Celery는 Broker를 거치지 않고, `delay()`를 호출하는 즉시 로컬 프로세스에서 함수를 실행합니다. (동기 실행)

```python
# test_settings.py
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True # Task 에러가 나면 테스트도 실패하게 함
```

이러면 Redis 없이도 로직 테스트가 가능합니다.

---

## 2. Mocking Tasks

Task를 호출하는 뷰(View)나 API를 테스트할 때, 실제로 Task를 실행할 필요는 없습니다.
"Task가 호출되었는가?"만 확인하면 됩니다.

```python
from unittest.mock import patch

def test_signup_view():
    with patch('tasks.send_welcome_email.delay') as mock_task:
        client.post('/signup', ...)
        
        # Task가 호출되었는지 확인
        mock_task.assert_called_with(user_id=1)
```

---

## 3. Testing Canvas (Integration Test)

Chain이나 Group 같은 복잡한 로직은 Eager Mode로 테스트하기 어렵습니다.
이때는 실제 Redis(또는 Mock Redis)를 붙여서 통합 테스트를 해야 합니다.

**Pytest-celery** 플러그인을 쓰면 테스트용 Celery App과 Worker를 띄워줍니다.

---

## 🛠️ Lab: Writing Tests

1.  `add` Task를 테스트하는 코드를 작성합니다.
2.  `override_settings(CELERY_TASK_ALWAYS_EAGER=True)` 데코레이터를 사용하여, `add.delay(1, 1)`이 즉시 2를 반환하는지 확인합니다.
3.  `patch`를 사용하여 Task 내부 로직은 실행하지 않고 호출 여부만 검증하는 테스트를 작성합니다.

---

## 📝 11주차 과제: Test Coverage 100%

**목표:** Week 6에서 만든 '이미지 처리 파이프라인(Chain)'을 테스트하세요.

1.  **Unit Test:** 각 Task (`download`, `resize`...)가 정상 동작하는지 개별 테스트.
2.  **Flow Test:** `chain`이 올바르게 구성되었는지 확인. (실행하지 말고 시그니처만 검사)
3.  **Mocking:** `download_image`가 실제 S3에 접속하지 않도록 `boto3`를 Mocking 하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Celery Task를 브로커 없이 로컬에서 동기적으로 즉시 실행하게 만드는 설정은? (task_always_eager)
2.  **Q2.** API 엔드포인트 테스트 시, 무거운 Celery Task가 실제로 실행되지 않게 하면서 호출 여부만 검증하기 위해 사용하는 Python 라이브러리는? (unittest.mock)
3.  **Q3.** `CELERY_TASK_EAGER_PROPAGATES = True` 설정의 역할은? (Eager 모드에서 Task 실행 중 발생한 예외를 호출자에게 전파하여 테스트가 실패하도록 함)

---

이제 모든 준비가 끝났습니다.
다음 주, 마지막 **Capstone Project**에서 여러분의 실력을 증명하세요.

**Test confident.**
