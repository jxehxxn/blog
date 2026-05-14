---
layout: post
title:  "Dreamhack 워게임 [wargame.kr] type confusion 풀이 — PHP 느슨한 비교(==) 한 줄로 끝내기"
date:   2026-05-14 23:15:00 +0900
categories: security web wargame dreamhack writeup php type-juggling
---

## 들어가며

이번 글은 Dreamhack의 클래식 PHP 워게임 **[wargame.kr] type confusion** (난이도 Bronze 2) 풀이입니다. 문제 페이지에 친절하게도 거의 모든 힌트가 적혀 있습니다.

> Simple Compare Challenge.
> hint? you can see the title of this challenge.
> :D

힌트가 곧 정답인 셈입니다. "type confusion" — 즉, PHP의 **느슨한 비교 연산자(`==`)** 가 자료형이 다른 두 값을 자기 마음대로 변환해서 비교하기 때문에 생기는, 정보보안 입문자가 PHP를 만나면 반드시 한 번은 거쳐야 하는 단골 취약점입니다.

스포일러부터 가면, 최종 익스플로잇은 `curl` 한 줄입니다.

```bash
curl -X POST "$SERVER/" --data-urlencode 'json={"key":true}'
# → {"code":true,"flag":"DH{c857ca841295d6df5f83186e647e3ef1f2a1530c}"}
```

이 한 줄이 왜 통하는지, 그리고 PHP의 `==`가 왜 이렇게 위험한지를 처음 보는 사람의 시선으로 차근차근 따라가 봅시다.

## 1. 첫인상 — 페이지 확인

VM이 부팅되면 단순한 HTML 페이지가 떨어집니다.

```html
<form onsubmit="return submit_check(this);">
  <input type="text" name="key" />
  <input type="submit" value="check" />
</form>
<a href="./?view-source">view-source</a>
```

입력란 하나, 그리고 친절하게 **`view-source` 링크**가 달려 있습니다. 워게임에서 `view-source`가 클릭 가능하면 그건 곧 "코드 읽어보고 풀어라"라는 의미죠.

## 2. 소스 코드 정독

`?view-source` 를 누르면 `show_source(__FILE__)`로 자신의 소스를 출력합니다.

```php
<?php
 if (isset($_GET['view-source'])) {
     show_source(__FILE__);
     exit();
 }
 if (isset($_POST['json'])) {
     usleep(500000);                          // ① 500ms 인위적 지연
     require("./lib.php");                    //    include for FLAG.
     $json = json_decode($_POST['json']);     // ② 사용자가 보낸 JSON 파싱
     $key  = gen_key();                       // ③ 서버가 매번 새 키 생성
     if ($json->key == $key) {                // ④ 핵심: 느슨한 비교 ==
         $ret = ["code" => true, "flag" => $FLAG];
     } else {
         $ret = ["code" => false];
     }
     die(json_encode($ret));
 }

 function gen_key(){
     $key = uniqid("welcome to wargame.kr!_", true);
     $key = sha1($key);
     return $key;
 }
?>
```

코드는 단 20여 줄. 보안적으로 중요한 4가지 포인트가 있습니다.

1. **`usleep(500000)`** — 매 요청마다 0.5초씩 지연. **무차별 대입(brute-force) 공격을 일부러 비싸게 만든다.** 즉, "키를 추측해서 맞춰서 푸는 풀이"는 의도된 길이 아니라는 강력한 신호.
2. **`gen_key()`** — `uniqid("welcome to wargame.kr!_", true)` 의 결과를 `sha1`. SHA1은 40자리 16진수 문자열로 항상 같은 형식: `^[0-9a-f]{40}$`. 즉 `$key`는 **항상 비어있지 않은 문자열**.
3. **`json_decode($_POST['json'])`** — 사용자 입력을 JSON으로 파싱. 두 번째 인자가 없으므로 객체(`stdClass`)로 반환. 객체 프로퍼티 `$json->key` 의 **자료형은 사용자가 보내기 나름**(string, number, bool, null, array, object 모두 가능).
4. **`$json->key == $key`** — 가장 위험한 한 줄. PHP의 `==`는 양변의 자료형이 다르면 **묵시적 형변환** 을 한다. 그 변환 규칙이 우리가 흔히 생각하는 직관과 어긋난다.

이 네 가지를 모두 본 순간, 풀이의 방향은 정해집니다: **`$json->key` 의 자료형을 의도적으로 조작해서, "어떤 sha1 문자열과도 `==` true가 되는 값"** 을 보낸다.

## 3. PHP 느슨한 비교(`==`)의 본질

PHP 7.x 기준 `==` 비교는 두 값의 자료형에 따라 아래처럼 동작합니다 (8.x에서 일부 규칙이 엄격해졌지만 이번 문제에 영향을 주는 부분은 동일).

| 좌변      | 우변      | 비교 방식                                       |
|----------|-----------|--------------------------------------------------|
| string   | string    | 문자열 비교 (단, 둘 다 numeric이면 숫자 변환 후 비교) |
| **bool** | **anything** | **양변을 bool로 변환 후 비교**                   |
| number   | string    | string을 number로 변환 후 비교                   |
| null     | anything  | 양변을 bool로 변환 후 비교                       |

핵심은 두 번째 줄입니다. **`bool == string`** 비교는 양변을 `bool`로 캐스팅합니다. 그리고 PHP에서 다음과 같은 변환 규칙이 있습니다.

- `(bool)""`          → `false`
- `(bool)"0"`         → `false`
- `(bool)"any non-empty, non-"0" string"` → **`true`**

따라서 `$key` 가 SHA1 결과(예: `"5e8d28f..."`) 처럼 비어있지 않은 문자열이라면 `(bool)$key === true` 입니다.

```php
true == "5e8d28f..."   // → 양변을 bool: true == true → TRUE !
```

서버는 매 요청마다 키를 다시 생성하지만, **종류가 무엇이든 항상 비어있지 않은 40자 문자열**이라는 점이 변하지 않습니다. 그 점이 결정적인 약점이 됩니다.

## 4. 익스플로잇

페이로드는 단 한 줄입니다.

```json
{"key": true}
```

JSON에서 `true`는 boolean 타입으로 디코딩되므로, `$json->key`는 PHP `bool(true)` 가 됩니다.

```bash
SERVER=http://<host>:<port>
curl -s -X POST "$SERVER/" --data-urlencode 'json={"key":true}'
```

응답:

```json
{"code":true,"flag":"DH{c857ca841295d6df5f83186e647e3ef1f2a1530c}"}
```

flag: `DH{c857ca841295d6df5f83186e647e3ef1f2a1530c}` ✓

### 4-1. 다른 방향의 답은 안 되는 이유

비교를 통과시키는 다른 후보들을 점검해보면 PHP의 변환 규칙이 더 또렷이 보입니다.

- `{"key": 0}` (정수 0)
  - `$key`는 SHA1, 즉 "0"으로 시작하지 않는 한 numeric 문자열이 아니다. → string 비교 → "0" == "5e8..." → false.
  - SHA1 결과가 `"0..."`이 아닌 한 통과 못 함. 운에 의존.
- `{"key": "0e123..."}` (소위 magic hash)
  - PHP 7에서 `"0eXXX" == "0eYYY"` 가 둘 다 numeric으로 0과 0이 되어 true가 되는 트릭. 하지만 우변이 매번 무작위 SHA1 이라 우변이 `0eXXX` 형태로 나올 확률이 매우 낮아 운에 의존.
- `{"key": [1,2,3]}` (배열)
  - PHP 7에서 `array == string` 은 항상 false. 통과 못 함.
- `{"key": null}`
  - `null == string` → string을 bool로 변환 → null은 false → false == true → false. 통과 못 함 (SHA1 결과가 비어있지 않은 한).

결국 **bool true** 만이 "어떤 SHA1 결과에 대해서도 항상 통과"하는 만능 페이로드입니다.

### 4-2. usleep이 의미하는 것

서버 코드는 `usleep(500000)`으로 매 요청마다 0.5초를 강제합니다. 만약 SHA1을 추측해서 맞히려고 한다면 한 번에 0.5초 × 2¹⁶⁰ 번을 시도해야 하므로 절대 불가능합니다. 이 한 줄이 의미하는 것은 출제자의 명시적 메시지입니다.

> "추측 말고, **타입을 가지고 놀아라**."

`usleep` 이 보일 때마다 "출제자는 brute-force 풀이를 막고 싶었다 → 다른 카테고리의 우회를 의도했다"라는 신호로 해석하는 습관을 들이면 좋습니다.

## 5. 취약점 해설 — Type Juggling

이 클래스를 일반화하면 다음과 같습니다.

> 서로 다른 자료형의 두 값을 비교할 때, 언어가 **묵시적 형변환** 을 수행하면, 공격자는 형변환 규칙의 빈틈을 노려 검사를 우회할 수 있다.

PHP뿐만 아니라 다른 동적 언어에서도 비슷한 패턴이 보입니다.

- **JavaScript**: `0 == "0"` (true), `[] == false` (true), `"" == 0` (true). `==` 대신 `===`를 쓰지 않으면 같은 부류의 문제 발생.
- **Python**: 비교 자체는 엄격하지만 `bool` 이 `int` 의 서브클래스라 `True == 1` (true). 비밀번호 검증에 boolean이 섞이면 위험.
- **Ruby**: `==` 이 클래스에 따라 재정의되며, 사용자 정의 객체에서 잘못 구현하면 동일한 취약점.

### 5-1. 위험성

이번 문제는 학습용 예제지만, 실무에서 같은 패턴이 등장하는 곳이 매우 많습니다.

- **세션 토큰 검증**: 서버가 만든 토큰과 사용자가 보낸 토큰을 `==` 로 비교 → JSON으로 토큰을 받았다면 type juggling 으로 우회.
- **CAPTCHA / OTP 검증**: 클라이언트가 보낸 코드와 서버 저장 코드를 `==`로. 같은 문제.
- **PHP unserialize 후 비교**: PHP 객체 비교(`==`)는 같은 클래스 + 같은 프로퍼티 값이어야 하지만, **`->key` 같은 동적 프로퍼티 접근 + json_decode** 처럼 자료형이 자유로워지는 곳은 같은 부류로 취급해야 한다.
- **API 키 / 라이선스 키 검증**: 사용자가 JSON으로 보낸 키와 DB의 키를 `==`로. 직격탄.

### 5-2. 올바른 방어

PHP에서 비밀 비교는 다음 두 가지 원칙을 항상 따라야 합니다.

1. **`===` (엄격한 비교) 사용**. 자료형이 다르면 무조건 false.

   ```php
   if ($json->key === $key) { ... }
   ```

   이 한 글자만 추가해도 이번 익스플로잇은 즉시 막힙니다. 단, `$json->key` 가 문자열이 아니라면 무조건 false가 되는 점을 의도적으로 활용하는 셈이 됩니다.

2. **상수 시간 비교** `hash_equals` 사용. 보안적으로 비교할 때는 타이밍 공격까지 막아야 합니다.

   ```php
   if (is_string($json->key) && hash_equals($key, $json->key)) { ... }
   ```

   `hash_equals` 는 두 문자열의 길이가 다르면 즉시 false, 같으면 모든 바이트를 비교해 상수 시간으로 동작합니다. CSRF 토큰, 세션 토큰, HMAC 검증 등에 반드시 이 함수를 써야 합니다.

3. **입력 검증을 비교 *이전*에**. 받은 값이 기대한 자료형(string, 길이, 형식)인지 먼저 검증한다.

   ```php
   if (!isset($json->key) || !is_string($json->key) || strlen($json->key) !== 40) {
       die(json_encode(["code" => false]));
   }
   if (hash_equals($key, $json->key)) { ... }
   ```

## 6. 정리 — 입문자가 가져갈 교훈

- **`==` 는 비교 연산자가 아니라 "거의 비교 연산자"** 다. PHP에서는 거의 항상 `===` 또는 `hash_equals` 를 써야 한다.
- 출제자가 일부러 넣어 둔 코드(`usleep`, `die(...)`, `random_bytes` 등)는 모두 "이쪽 길로 가지 말라"는 안내판이다. 그 안내판을 읽으면 정답 경로가 보인다.
- 비밀 값이 항상 같은 "형태"(예: SHA1 → 비어있지 않은 40자 hex)를 가진다면, 그 형태에서 **항상 true로 평가되는 형변환 짝(예: bool true)** 이 존재하는지 점검하자.
- `json_decode` 는 공격자에게 **자료형의 선택권** 을 준다. PHP에서 JSON을 받은 직후에는 항상 자료형을 명시적으로 검증해야 한다.

PHP의 type juggling 은 한 번 손에 익으면 여러 문제에서 반복적으로 만나는 클래식입니다. 비슷한 부류의 다음 단계로 `simple_sqli`(SQLi + magic hash) 나 `php-1`(`strcmp` 우회 + 배열 인자) 같은 문제를 풀어보면 패턴이 머리에 박힙니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 "파일 다운로드" 버튼을 클릭했는데 ZIP이 떨어지지 않음

문제 페이지의 "문제 파일 받기" 버튼을 클릭했지만 `~/Downloads/` 에 새 파일이 추가되지 않았습니다. 잠시 당황했는데, 곧 페이지의 `view-source` 링크와 출제자 노트(`hint? you can see the title of this challenge.`)를 보고 **이 문제는 별도 첨부파일이 없고, 라이브 서버의 `?view-source` 자체가 소스 공급원** 임을 깨달았습니다.

**회고**: 모든 Dreamhack 문제가 ZIP을 제공하지는 않습니다. 라이브 페이지에 `view-source`, `show_source`, `phpinfo` 같은 명시적 노출 경로가 있으면 그 자체가 의도된 소스 채널입니다. 다음부터는 "버튼 → ZIP" 의 단계로 가기 전에 `/`, `/source`, `/?view-source`, `/index.php?view-source` 등을 먼저 두드려본 뒤 ZIP 시도를 합니다.

### 실패 2. 첫 추측이 "magic hash" 였음

문제 제목 `type confusion`을 보고 처음 떠올린 것은 PHP의 **magic hash**(`"0e..."` 형태) 트릭이었습니다. `{"key":"0e0"}` 같은 페이로드를 잠깐 고민했지만, 곧 `$key`가 매 요청마다 새 SHA1 이라 우변이 **운 좋게 `"0e..."` 형태가 될 확률** 에 의존해야 함을 깨닫고 폐기했습니다.

**회고**: type juggling 패밀리는 1) `bool` 대상의 `(bool)$x` 변환, 2) numeric 문자열의 `0eXXX` magic hash, 3) array vs string 의 항상-false 트릭, 세 가지가 있습니다. 결정 트리는 다음과 같이 짚으면 빠릅니다.

- 우변이 **항상 비어있지 않은 문자열** 인가? → `{"key": true}` 한 방.
- 우변이 **운 좋게 `0eXXX` 형태로 나올 수 있는 해시** 인가? (예: MD5 충돌, SHA-256 충돌) → magic hash 시도 (운에 의존).
- 좌변이 배열을 거부하지 않는가? → `array == string` 의 PHP 7 ↔ 8 차이 활용.

이번 문제는 첫 번째 케이스라 사실 5초 컷이 가능했습니다.

### 실패 3. PHP 버전을 추측하지 않고 곧장 페이로드를 시도

PHP 8에서는 `==`의 일부 규칙이 엄격해졌습니다(특히 `0 == "abc"` 가 더 이상 true가 아님). 만약 출제자가 PHP 8 + 추가 검증을 했다면 단순 bool 트릭이 통하지 않을 수도 있었습니다. 다행히 이번 서버는 PHP 7.x 동작과 호환되는 환경이었지만, 한 번 헛수고 했다면 30초는 더 썼을 것입니다.

**회고**: 응답 헤더의 `Server: Apache/2.4.18 (Ubuntu)` 같은 정보, 그리고 `phpinfo()` 가 노출돼 있다면 PHP 버전을 먼저 확인하는 습관이 필요합니다. 비교 트릭은 버전 의존성이 있으므로 항상 그 환경에서 어떤 형변환 규칙이 적용되는지 알고 들어가야 합니다.
