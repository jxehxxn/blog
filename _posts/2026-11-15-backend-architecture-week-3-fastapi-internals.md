---
layout: post
title: "Backend Architecture Mastery: Week 3 - FastAPI Internals"
---

FastAPI가 Flask나 Django보다 빠르고 생산성이 높은 이유는 단순히 "비동기라서"가 아닙니다. 그 내부에는 **Pydantic**이라는 강력한 데이터 엔진과, **Dependency Injection (DI)**이라는 우아한 디자인 패턴이 자리 잡고 있습니다.

오늘은 FastAPI의 심장을 해부해 봅니다.

---

## 1. Pydantic v2: Rust의 힘을 빌리다

FastAPI는 데이터를 주고받을 때 `Pydantic` 모델을 사용합니다.
Pydantic v2부터는 핵심 코어가 **Rust**로 재작성되었습니다. 이로 인해 검증 속도가 비약적으로 빨라졌습니다.

### Type Hints are not just Hints
Python의 타입 힌트는 원래 "장식"에 불과했습니다. 런타임에는 무시되죠.
하지만 Pydantic은 이 타입 힌트를 **실제 검증 규칙**으로 사용합니다.

```python
from pydantic import BaseModel, Field, EmailStr

class UserSignup(BaseModel):
    username: str = Field(min_length=3)
    email: EmailStr
    age: int = Field(gt=0, le=150)
```

이 코드는 단순한 클래스 정의가 아닙니다. JSON 파싱, 타입 캐스팅, 데이터 검증, 에러 메시지 생성을 모두 수행하는 **Validator**입니다.

---

## 2. Dependency Injection: FastAPI의 꽃

보통 "의존성 주입"이라고 하면 Spring Framework의 거대한 컨테이너를 떠올립니다. 어렵고 복잡하죠.
하지만 FastAPI의 DI 시스템(`Depends`)은 놀라울 정도로 직관적이고 강력합니다.

### 왜 쓰는가? (The "Why")
1.  **코드 중복 제거:** DB 세션 생성, 사용자 인증 로직을 매 핸들러마다 쓸 필요가 없습니다.
2.  **테스트 용이성:** 실제 DB 대신 Mock DB를 주입하기 쉽습니다.
3.  **계층 분리:** 비즈니스 로직과 인프라 로직을 분리합니다.

### 어떻게 동작하는가? (The "How")

```python
from fastapi import Depends, Header, HTTPException

# 1. 의존성 정의 (함수든 클래스든 상관없음)
async def verify_token(x_token: str = Header(...)):
    if x_token != "secret-token":
        raise HTTPException(status_code=400, detail="Token Invalid")
    return x_token

# 2. 의존성 주입
@app.get("/items")
async def read_items(token: str = Depends(verify_token)):
    # 이 함수는 verify_token이 성공해야만 실행됩니다.
    return {"token": token}
```

FastAPI는 요청이 들어오면 `read_items`의 파라미터를 분석합니다. `Depends`를 발견하면 `verify_token`을 먼저 실행하고, 그 결과를 `token` 변수에 넣어줍니다. 이 과정은 재귀적으로 일어납니다. (의존성의 의존성의 의존성...)

---

## 3. Middleware vs Dependencies

흔한 질문입니다. "로그인은 미들웨어에서 하나요, Dependency에서 하나요?"

*   **Middleware:** 요청이 앱에 들어오고 나가는 **전체 파이프라인**을 감쌉니다. (CORS, GZip, 전역 로깅)
*   **Dependency:** 특정 **엔드포인트**에 필요한 데이터를 준비합니다. (DB 세션, 사용자 인증, 권한 체크)

**Best Practice:** 사용자 인증(Authentication)은 미들웨어보다 **Dependency**에서 하는 것이 좋습니다. Swagger UI(OpenAPI)와 자동으로 연동되고, 특정 라우터마다 다른 인증 방식을 적용하기 유연하기 때문입니다.

---

## 🛠️ Lab: Building a Clean Architecture

DI를 이용해 Service Layer 패턴을 흉내 내어 봅시다.

1.  **Repository:** DB에 직접 접근하는 계층
2.  **Service:** 비즈니스 로직을 처리하는 계층
3.  **Router:** HTTP 요청을 받고 응답하는 계층

```python
# deps.py
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# service.py
class UserService:
    def __init__(self, db: Session):
        self.db = db
    
    def create_user(self, user: UserCreate):
        # 복잡한 가입 로직...
        return new_user

# router.py
@app.post("/users")
def create_user(
    user: UserCreate, 
    db: Session = Depends(get_db)
):
    service = UserService(db) # 수동 주입 (아직은)
    return service.create_user(user)
```
*Tip: `UserService` 자체도 Depends로 만들면 더 깔끔해집니다.*

---

## 📝 3주차 과제: Advanced Dependency

**목표:** 다음 요구사항을 만족하는 Custom Dependency를 만드세요.

1.  **PaginationDependency:** 쿼리 파라미터로 `skip` (default 0), `limit` (default 10, max 100)을 받습니다.
2.  **UserAgentDependency:** 헤더에서 `User-Agent`를 읽어옵니다. 모바일 기기인지 아닌지 판별하는 로직을 포함하세요.
3.  위 두 의존성을 동시에 사용하는 API 엔드포인트를 작성하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Pydantic 모델에서 선언된 필드 타입과 다른 타입의 데이터가 들어오면(예: int 자리에 "123" 문자열), FastAPI는 어떻게 반응하나요? (자동 변환 시도 or 에러?)
2.  **Q2.** `Depends`를 사용할 때 함수 실행 결과를 캐싱하여, 한 요청 내에서 여러 번 호출되어도 한 번만 실행되게 하려면 어떻게 해야 하나요? (기본 동작임)
3.  **Q3.** `yield`를 사용하는 Dependency의 주 목적은 무엇인가요? (리소스 정리/Teardown)

---

다음 주에는 요청에 대한 응답을 보낸 **후**에 무언가를 처리하는 **Background Tasks**에 대해 알아봅니다. 비동기 처리의 서막입니다.

**Inject Wisely.**
