---
layout: post
title: "Backend Architecture Mastery: Week 8 - Event-Driven Architecture (Kafka vs RabbitMQ)"
---

"RabbitMQ랑 Kafka랑 뭐가 달라요? 둘 다 메시지 큐 아닌가요?"

면접 단골 질문이자, 아키텍처를 설계할 때 가장 많이 하는 고민입니다.
결론부터 말하면, 둘은 **철학**이 다릅니다.

*   **RabbitMQ:** 우체부. 편지를 배달하면(Ack), 가방에서 비웁니다(Delete).
*   **Kafka:** 도서관/로그(Log). 책을 빌려가도(Read), 책장에 책은 그대로 있습니다(Retention).

오늘은 이 차이가 만드는 아키텍처의 변화, **Event-Driven Architecture (EDA)**를 배웁니다.

---

## 1. Pub/Sub Pattern

전통적인 Request-Response 패턴은 강한 결합(Tight Coupling)을 만듭니다.
"주문 서비스"가 "배송 서비스"의 API를 직접 호출해야 하죠. 배송 서버가 죽으면 주문도 못 받습니다.

EDA에서는 **Event(사건)**를 발행합니다.
1.  주문 서비스: "주문이 발생했음! (OrderCreated)" -> 이벤트 버스에 소리침.
2.  배송 서비스: (귀를 기울이다가) "어? 주문 왔네? 배송 준비해야지."
3.  이메일 서비스: (귀를 기울이다가) "어? 주문 왔네? 메일 보내야지."

주문 서비스는 누가 듣고 있는지 모릅니다. 그냥 소리치고 끝입니다. (Decoupling)

---

## 2. RabbitMQ vs Kafka: The Showdown

| 특징 | RabbitMQ (Smart Broker) | Kafka (Smart Client) |
| :--- | :--- | :--- |
| **메시지 보관** | 소비되면 삭제됨 (Queue) | 설정한 기간(예: 7일) 동안 저장됨 (Log) |
| **처리 방식** | 1:1 처리에 강함 (Task Queue) | 1:N 방송에 강함 (Streaming) |
| **성능** | 수만 TPS | 수십~수백만 TPS |
| **복잡도** | 비교적 쉬움 | 운영 난이도 높음 (Zookeeper 등) |
| **용도** | **작업 처리** (이메일 발송, 인코딩) | **데이터 파이프라인** (로그 수집, 클릭 추적) |

**요약:**
*   "이거 처리해 주세요" (Command) -> **RabbitMQ**
*   "이런 일이 있었어요" (Event) -> **Kafka**

---

## 3. Event Consistency & Ordering

Kafka를 쓸 때 가장 주의할 점은 **순서**입니다.
주문 생성(Created) -> 결제 완료(Paid) -> 배송 시작(Shipped) 순서로 이벤트가 와야 하는데, 네트워크 문제로 `Paid`가 `Shipped`보다 늦게 오면?

Kafka는 **Partition** 내에서만 순서를 보장합니다. 따라서 동일한 `Order ID`를 가진 이벤트는 반드시 동일한 파티션으로 가도록 **Partition Key**를 잘 설정해야 합니다.

---

## 🛠️ Lab: Simple Pub/Sub with RabbitMQ Fanout

RabbitMQ로도 Pub/Sub을 흉내 낼 수 있습니다. `Fanout Exchange`를 쓰면 됩니다.

1.  **Producer:** `logs`라는 Fanout Exchange에 메시지를 쏘기만 합니다. (Queue 이름 안 씀)
2.  **Consumer A (Logging):** 큐를 하나 만들어서 `logs`에 바인딩. 모든 메시지를 파일에 저장.
3.  **Consumer B (Alert):** 큐를 하나 만들어서 `logs`에 바인딩. "Error" 단어가 있으면 슬랙 알림.

하나의 메시지가 복제되어 A, B 두 곳으로 동시에 배달되는 것을 확인하세요.

---

## 📝 8주차 과제: Architecture Design

**목표:** "배달의 민족" 같은 배달 앱의 주문 시스템 아키텍처를 설계하여 다이어그램(Draw.io 등)과 텍스트로 제출하세요.

**요구사항:**
1.  사용자가 주문을 하면 **RabbitMQ**를 통해 주문 접수 처리가 됩니다.
2.  주문이 완료되면 **Kafka**에 `OrderCompleted` 이벤트를 발행합니다.
3.  **Sub 1:** 사장님 앱에 "주문!" 알림이 뜹니다.
4.  **Sub 2:** 데이터 분석팀이 주문 정보를 하둡(Hadoop)에 저장합니다.
5.  왜 RabbitMQ와 Kafka를 섞어서 썼는지 이유를 서술하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Kafka에서 데이터를 저장하는 파일 단위이자, 병렬 처리의 단위는 무엇인가요? (Partition)
2.  **Q2.** RabbitMQ의 Exchange 타입 중, 연결된 모든 큐에 무조건 메시지를 뿌리는 타입은? (Fanout)
3.  **Q3.** 이벤트를 발행하는 주체가 구독자가 누구인지 알 필요가 없는 성질을 무엇이라 하나요? (Decoupling / Loose Coupling)

---

다음 주, 우리는 데이터베이스로 눈을 돌립니다. 수천 명이 동시에 재고를 차감하려고 할 때, DB는 어떻게 정합성을 유지할까요? **Concurrency Control**의 세계로 갑니다.

**Publish and Subscribe.**
