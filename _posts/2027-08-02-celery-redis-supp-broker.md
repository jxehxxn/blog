---
layout: post
title: "Distributed Task Processing with Celery & Redis: Supplement - Message Broker Concepts (Redis vs RabbitMQ)"
---

Week 1에서 "Redis를 브로커로 쓴다"고 했습니다. 그런데 구글링을 해보면 "RabbitMQ가 진짜 브로커고 Redis는 야매(?)다"라는 글도 보입니다.
정확히 무엇이 다르고, 언제 무엇을 써야 할까요?

이 보충 강의에서는 Celery의 두 양대 산맥인 Redis와 RabbitMQ를 비교 분석합니다.

---

## 1. Redis as a Broker

Redis는 원래 **In-Memory Key-Value Store**입니다. 메시지 큐 전용 소프트웨어가 아닙니다.
하지만 `List` 자료구조(`LPUSH`, `BRPOP`)와 `Pub/Sub` 기능이 워낙 강력해서 브로커로도 훌륭하게 동작합니다.

### 장점 (Pros)
*   **쉽다:** 개발자라면 누구나 Redis를 압니다. 설치도 쉽고 관리도 쉽습니다.
*   **빠르다:** 메모리 기반이라 속도가 깡패입니다.
*   **다용도:** 브로커로 쓰면서 동시에 캐시(Cache), 세션 저장소로도 쓸 수 있습니다.
*   **Visibility:** `LRANGE` 명령어로 큐에 뭐가 들어있는지 눈으로 보기 쉽습니다.

### 단점 (Cons)
*   **신뢰성:** 메모리 기반이라 전원 나가면 데이터 날아갑니다. (Persistence 설정으로 보완 가능하지만 완벽하진 않음)
*   **메모리 제한:** 큐에 메시지가 수백만 개 쌓이면 메모리가 터집니다(OOM).
*   **복잡한 라우팅 불가:** RabbitMQ처럼 "이 메시지는 A큐와 B큐에 동시에 복사해줘" 같은 고급 라우팅이 어렵습니다.

---

## 2. RabbitMQ as a Broker

RabbitMQ는 **AMQP(Advanced Message Queuing Protocol)**라는 표준을 구현한 **전문 메시지 브로커**입니다. 태생부터 큐를 위해 태어났습니다.

### 장점 (Pros)
*   **신뢰성 (Reliability):** 메시지 영속성(Persistence), 전달 보장(Delivery Acknowledgement)이 확실합니다.
*   **유연한 라우팅 (Exchange):** Direct, Fanout, Topic, Headers 등 다양한 방식으로 메시지를 분배할 수 있습니다.
*   **메모리 관리:** 메시지가 많아지면 디스크로 내립니다. 메모리 부족으로 죽지 않습니다.

### 단점 (Cons)
*   **어렵다:** Exchange, Queue, Binding, Routing Key 같은 개념을 배워야 합니다.
*   **운영 비용:** Erlang 기반이라 운영 노하우가 좀 필요합니다.

---

## 3. When to use what?

### Use Redis if:
*   설정이 간편하고 빠른 시작이 필요하다.
*   메시지 유실이 좀 있어도 치명적이지 않다. (예: 조회수 카운트, 캐시 갱신)
*   단기 프로젝트거나 규모가 크지 않다.
*   이미 기술 스택에 Redis가 있다.

### Use RabbitMQ if:
*   메시지는 절대 잃어버리면 안 된다. (예: 결제 처리, 주문 생성)
*   복잡한 라우팅이 필요하다. (예: 하나의 이벤트를 여러 서비스가 구독)
*   큐에 메시지가 엄청나게 쌓일 수 있다.
*   엔터프라이즈급 안정성이 필요하다.

**Celery 공식 권장:** 프로덕션 환경에서는 **RabbitMQ**를 권장하지만, 많은 스타트업과 중소규모 서비스는 **Redis**로도 충분히 잘 운영하고 있습니다. 우리 강의에서는 접근성이 좋은 Redis를 메인으로 사용하지만, RabbitMQ로 전환하는 법도 다룰 것입니다.

---

## 4. Redis Queue Under the Hood

Celery가 Redis를 쓸 때 내부적으로 어떻게 동작할까요?

1.  **Task 넣기:** `LPUSH celery "task_json_string"`
    *   `celery`라는 이름의 리스트 왼쪽에 넣습니다.
2.  **Task 가져오기:** `BRPOP celery 0`
    *   워커는 `celery` 리스트의 오른쪽에서 꺼내옵니다.
    *   `B`는 Blocking입니다. 리스트가 비어있으면 데이터가 들어올 때까지 대기합니다. (Polling 아님!)

이 간단한 메커니즘 덕분에 Redis가 브로커 역할을 할 수 있는 것입니다.
