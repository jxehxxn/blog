---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 6 - Workflow Orchestration (Canvas: Chain, Group, Chord)"
---

단순히 "이거 해"가 아니라, **"이거 다 하고, 저거 하고, 그 다음에 이거 해"**를 하고 싶을 때가 있습니다.
Celery의 **Canvas(Workflow)** 기능은 분산 작업을 예술(Art)로 만듭니다.

---

## 1. Chain: 순서대로 실행하기 (Link)

Task A의 결과가 Task B의 인자로 들어가야 할 때 씁니다. (`|` 연산자)

```python
from celery import chain
from tasks import add, mul

# (4 + 4) * 8
res = chain(add.s(4, 4), mul.s(8))()
res.get() # 64
```
*   `.s()`: Signature. 작업을 정의만 하고 실행은 안 함.
*   `add(4, 4)`의 결과인 8이 `mul(8)`의 **첫 번째 인자**로 자동 전달됨 -> `mul(8, 8)` 실행.

---

## 2. Group: 동시에 실행하기 (Parallel)

여러 작업을 병렬로 던지고 결과를 한 번에 받고 싶을 때 씁니다.

```python
from celery import group

# 10개의 덧셈을 동시에
job = group(add.s(i, i) for i in range(10))
res = job.apply_async()
res.get() # [0, 2, 4, ..., 18]
```

---

## 3. Chord: MapReduce (분산 처리 후 취합)

가장 강력한 패턴입니다.
**Group으로 뿌리고(Map), 다 끝나면 Callback을 실행(Reduce)합니다.**

```python
from celery import chord

# 1. (i + i) 10번 실행 (Header)
# 2. 다 끝나면 결과를 다 합침 (Body - xsum)
res = chord((add.s(i, i) for i in range(10)), xsum.s())()
res.get() # 90
```
주의: Chord를 쓰려면 **Result Backend**가 필수입니다.

---

## 4. Starmap & Chunks

*   **Chunks:** 작업이 100만 개면 큐에 넣는 데만 한 세월입니다. 100개씩 묶어서(Chunk) 하나의 Task로 만듭니다. 오버헤드가 획기적으로 줍니다.
    ```python
    add.chunks(zip(range(100), range(100)), 10).apply_async()
    ```

---

## 🛠️ Lab: Image Processing Pipeline

사용자가 업로드한 이미지를 처리하는 파이프라인을 Canvas로 구현해 봅니다.

1.  **Tasks:**
    *   `download_image(url)` -> `path` 반환
    *   `resize_image(path)` -> `path` 반환
    *   `upload_s3(path)` -> `s3_url` 반환
    *   `send_email(s3_url)`

2.  **Workflow:**
    ```python
    workflow = chain(
        download_image.s(url),
        resize_image.s(),
        upload_s3.s(),
        send_email.s(user_email)
    )
    workflow.apply_async()
    ```

---

## 📝 6주차 과제: MapReduce Word Count

**목표:** 거대한 텍스트 파일을 분산 처리하여 단어 수를 세는 Chord를 구현하세요.

1.  **Split Task:** 텍스트를 1000줄씩 쪼개서 리턴.
2.  **Count Task (Map):** 쪼개진 텍스트를 받아 단어 빈도수(Dict)를 계산.
3.  **Sum Task (Reduce):** 여러 개의 Dict를 받아서 하나로 합침.
4.  이것들을 `chord`로 연결하여 실행하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Task A의 실행 결과가 Task B의 입력으로 전달되도록 연결하는 Canvas Primitives는? (Chain)
2.  **Q2.** 여러 개의 Task를 병렬로 실행하고, 모든 Task가 완료될 때까지 기다렸다가 그 결과를 모아서 후속 Task(Callback)를 실행하는 패턴은? (Chord)
3.  **Q3.** 대량의 작은 Task들을 묶어서 한 번에 처리함으로써 큐잉 오버헤드를 줄이는 기능은? (Chunks)

---

다음 주, "매일 아침 9시에 실행해줘."
주기적인 작업을 위한 스케줄러 **Celery Beat**를 만납니다.

**Chain reaction.**
