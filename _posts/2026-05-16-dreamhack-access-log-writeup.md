---
layout: post
title:  "Dreamhack 워게임 access-log 풀이 — sqlmap 의 blind SQLi 페이로드를 역추적해 dvwa.flag 복원하기"
date:   2026-05-16 12:00:00 +0900
categories: security web wargame dreamhack writeup forensics sqli blind-sqli
---

## 들어가며

Dreamhack 웹해킹 워게임 **access-log** (난이도 Bronze 2) 풀이입니다. **공격자 시점이 아니라 디펜더 시점** — `access.log` 한 파일만 보고 공격자가 sqlmap 으로 무엇을 빼냈는지 복원하는 사이버보안 입문 정통 forensics 문제.

문제 설명:

> 당신은 이번 해킹 사건의 분석을 담당하게 되었습니다. 그런데 access.log 파일 하나만이 제공 가능하다고 합니다. 과연 분석을 마치고 플래그를 찾아낼 수 있을까요?

스포일러로 결론: 공격자(`sqlmap/1.2.4`) 가 DVWA 의 `/vulnerabilities/sqli/` 에 시간 기반 blind SQLi 를 쏴서 `dvwa.flag` 테이블의 `value` 컬럼을 한 글자씩 추출했습니다. **그 binary search 과정이 access.log 의 URL 들에 그대로 박혀** 있어, 우리가 같은 페이로드를 거꾸로 읽으면 추출된 문자열을 복원할 수 있습니다.

```python
# 핵심 한 줄: 매 position 의 != 비교 → 그 N 이 그 자리의 ASCII
chars[pos] = first('!=' comparison's num at this position)
result = ''.join(chr(chars[pos]) for pos in sorted(chars))
# → 'DH{anA1yz1nGVe3yB19L0g}'
```

flag: `DH{anA1yz1nGVe3yB19L0g}` — "analyzing very big log" 의 leet 표기.

> Note: 사이버보안 입문 학습용 PoC. 실제 사건에 대한 분석은 적법한 권한과 절차 하에서.

## 1. 자료 둘러보기

```
$ wc -l access.log
6655 access.log
```

6,655 줄. 처음 몇 줄을 보면:

```
172.20.0.1 - - [26/Apr/2024:17:38:09 +0000] "GET /vulnerabilities/sqli/?id=1&Submit=Submit HTTP/1.1" 200 1686 "-" "sqlmap/1.2.4#stable (http://sqlmap.org)"
```

명백한 단서들:

- IP 단일: `172.20.0.1`
- UA: 6,649 줄이 `sqlmap/1.2.4#stable` (공격 자동화 도구). 2 줄만 Mozilla(수동 테스트), 4 줄은 `-` (timeout 408).
- 타깃: `/vulnerabilities/sqli/` (DVWA SQLi 페이지) 6,650 줄 + `/vulnerabilities/sqli_blind/` 1 줄.

즉 **DVWA 인스턴스에 sqlmap 으로 SQLi 자동 공격**.

## 2. 무엇을 빼냈는가 — SELECT 패턴 인덱싱

sqlmap 의 모든 페이로드 URL 안에는 **실제 실행되는 SQL 조각** 이 박혀있다. URL 디코드 후 `SELECT ... FROM ...` 패턴을 카운트해보면 attacker 의 우선순위가 보인다.

```python
import re, urllib.parse, collections
queries = collections.Counter()
for line in open('access.log'):
    url = urllib.parse.unquote(line)
    for sq in re.findall(r'SELECT [^,)]{1,200}', url):
        queries[sq] += 1
for q, c in queries.most_common(15):
    print(f'{c:5d}  {q[:80]}')
```

상위 결과:

```
 1363  SELECT IFNULL(CAST(password AS CHAR
 1004  SELECT IFNULL(CAST(avatar AS CHAR
  820  SELECT IFNULL(CAST(last_login AS CHAR
  626  SELECT IFNULL(CAST(column_name AS CHAR
  527  SELECT IFNULL(CAST(table_name AS CHAR
  243  SELECT IFNULL(CAST(`user` AS CHAR
  222  SELECT IFNULL(CAST(`value` AS CHAR    ← ★
  ...
```

**`value` 컬럼** 이 222번 등장한다. 그리고 같은 URL 들에 `FROM dvwa.flag` 가 있다. 즉 sqlmap 은 **dvwa.flag.value** 를 한 글자씩 빼고 있었다. flag 가 그 안에 있다.

## 3. 시간 기반 Blind SQLi 의 페이로드 구조

sqlmap 이 쏘는 한 줄의 형태:

```sql
1' AND 9804=IF(
  (ORD(MID(
     (SELECT IFNULL(CAST(`value` AS CHAR), 0x20)
        FROM dvwa.flag
        ORDER BY <col>
        LIMIT <row>,1
   ), <pos>, 1)) > <N>
  ),
  SLEEP(1),
  9804
)-- iSDD
```

해석:

- `LIMIT <row>,1` — 추출 대상 행. dvwa.flag 는 보통 행이 하나(`LIMIT 0,1`).
- `<pos>` — 추출하려는 문자열 위치 (1-indexed).
- `>N` — 그 위치의 char ASCII 값이 N 보다 큰지 묻는 비교.
- 참이면 `SLEEP(1)` 발동 → 응답 1초 지연.

**서버가 1초 지연 → 그 위치 char > N 이 TRUE**.

sqlmap 은 이 한 비트씩 binary search 로 좁혀가다가, 정확한 값 N 을 찾으면 `!=N` 으로 확정. `!=N` 이 FALSE (sleep 안 발동) 이면 char == N.

### 3-1. log 에는 응답 시간이 없는데 어떻게 알지?

여기가 핵심 통찰. log 에는 응답 시간이 안 박혀 있지만, **`!=N` 조건 자체가 sqlmap 의 마지막 confirmation step** 이므로, 코드 상에서 그 한 줄이 등장한다는 것은 곧 "이전 binary search 가 N 으로 수렴했다" 는 의미.

즉, sleep 발동 여부를 측정하지 않아도, **각 position 의 `!=N` 비교에서 N 을 읽으면 그 자리의 ASCII** 다.

## 4. 페이로드 복원

```python
import re, urllib.parse, collections

queries = []
for line in open('access.log'):
    url = urllib.parse.unquote(line)
    if 'dvwa.flag' not in url: continue
    m = re.search(
        r'CAST\(`?value`? AS CHAR\).*?LIMIT (\d+),1\),(\d+),1\)\)([!=><]+)(\d+)',
        url)
    if m:
        row, pos, op, num = m.groups()
        queries.append((int(row), int(pos), op, int(num)))

# 각 position 의 첫 `!=` 비교가 그 자리의 ASCII
by_pos = collections.defaultdict(list)
for row, pos, op, num in queries:
    by_pos[pos].append((op, num))

result = ''
for pos in sorted(by_pos):
    eqs = [n for op, n in by_pos[pos] if op == '!=']
    result += chr(eqs[0]) if eqs else '\x00'

print(repr(result))
# → 'DH{anA1yz1nGVe3yB19L0g}\x00'
print(result.replace('\x00',''))
# → DH{anA1yz1nGVe3yB19L0g}
```

23 글자 복원 + 1 글자는 '\x00' (= 종료 표시. ORD(NULL byte) 0, `>1`,`>32`,`>64` 모두 FALSE 였음).

flag: `DH{anA1yz1nGVe3yB19L0g}` ✓

## 5. 보안적 관점 — 무엇을 배워가나

### 5-1. Time-based blind SQLi 의 본질

이 클래스는 **에러 메시지도 데이터 응답도 안 보여주는 가장 어려운 SQLi** 지만, "응답 시간" 이라는 1 비트 채널이 있으면 충분히 데이터를 빼낼 수 있다.

- 한 글자 (8 비트) 추출에 평균 8 회 요청. ASCII 영문 + 숫자 + 일부 기호 (0~127) 라면 7 회.
- 22 글자 flag = 약 150-200 회 요청. log 의 222 회와 거의 일치.

### 5-2. 디펜더 입장의 단서

이 문제가 보여주는 것: **공격 페이로드 자체가 가장 강력한 증거**. 응답 시간 / 응답 본문이 없어도, 페이로드의 `binary search 패턴` 만으로 attacker 가 무엇을 빼냈는지 복원 가능.

실무 디펜더가 챙겨야 할 패턴:

- 같은 URL 에 **다양한 `<N>` 값** 으로 매우 비슷한 쿼리들이 수백~수천 번 — blind SQLi 시그니처.
- UA 가 `sqlmap`/`nmap`/`nikto`/`gobuster` 같이 알려진 도구 — 즉각 alert.
- 응답 코드 408 (timeout) 이 SLEEP-based 페이로드와 함께 등장 — SQLi 가 실제로 sleep 을 발동시켰다는 증거.

### 5-3. 같은 카테고리의 실무 사례

- **PHPMyAdmin 로그 분석** : 공격자가 information_schema 를 enumerate 한 흔적.
- **WAF false-negative 분석** : 페이로드가 WAF 를 우회한 흔적을 추후 분석.
- **사고 대응 (DFIR)** : 침입자의 행위를 access log + 시스템 로그 + DB 로그 등 다층으로 재구성.

### 5-4. 어떻게 막았어야 하나

1. **Prepared statements** — `id` 파라미터를 `?` placeholder 로. DVWA 의 `low` 시큐리티 레벨이 SQLi 의 교과서 예시.
2. **WAF + Rate limit** — sqlmap 처럼 초당 수십 회 비슷한 페이로드 보내면 즉시 차단.
3. **Application logging** : 의심스러운 SQL 키워드 (`UNION`, `SLEEP`, `INFORMATION_SCHEMA`) 가 query 에 박힌 모든 요청을 별도로 alert.
4. **DB user 권한 최소화** — DVWA app user 가 `INFORMATION_SCHEMA` 를 못 보게.

## 6. 정리 — 입문자가 가져갈 교훈

- **Forensics 의 1번 원칙: raw log 안에 모든 단서가 있다**. 응답 시간 없이도, 페이로드 자체에서 binary search 패턴을 읽으면 추출 데이터 복원 가능.
- sqlmap 의 blind SQLi 페이로드는 **매우 규칙적**. `>` 비교로 좁혀가다 `!=` 로 확정. 이 패턴 한 번 기억해두면 평생 쓴다.
- **URL 디코드 + 정규식** 만 알면 6,655 줄 log 가 5초 안에 분석된다.
- 디펜더가 attacker 의 페이로드 패턴 (sqlmap signature) 을 알아두면 SOC 알람 룰을 정확하게 짤 수 있다.

같은 카테고리의 다음 단계로 **weblog-1** (이미 풀음 — 이 시리즈의 step 1) 과 access-log Advanced, 그리고 일반적인 DFIR/SOC 분석 문제들을 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. flag 가 log 안에 plaintext 로 있을 거라 가정

처음에 `grep 'DH{' access.log` 한 번. 0 hit. URL-encoded `DH%7B` 도 0 hit. 잠시 멈춤.

**회고**: 로그 안 flag 가 raw 로 박혀있는 경우는 드물다. 특히 "사고 분석" 류 문제는 항상 **공격자가 빼낸 데이터** 가 flag. raw grep 이 0 이면 즉시 "공격자가 빼낸 내용을 페이로드로 복원" 단계로 넘어가자.

### 실패 2. 응답 시간이 없어서 blind SQLi 복원이 불가능하다고 잘못 결론

처음에 "시간 기반 blind SQLi 인데 access.log 에는 응답 시간이 없네... 어떻게 복원하지?" 하고 막혔다. 그러다 페이로드의 마지막에 등장하는 **`!=N` 비교** 가 사실은 sqlmap 의 confirmation step 이라는 점을 인지. binary search 가 N 으로 수렴한 뒤 마지막에 `!=N` 으로 한 번 더 확인하는 단계.

**회고**: 도구의 동작 순서를 알면 "남는 흔적" 도 시간 정보 없이 의미를 가진다. sqlmap 의 blind SQLi 알고리즘 :
1. `>N` 으로 bisect
2. 수렴 후 `=N` 또는 `!=N` 으로 confirm

이 두 단계를 기억해두면 access log 에서 timing 없이도 추출 데이터 복원 가능.

### 실패 3. 처음 정규식이 너무 좁아서 일부 패턴 누락

처음 `SELECT ... FROM dvwa.flag ... > N` 매칭에 `LIMIT` 부분 정규식을 너무 좁게 짜서 일부 페이로드 매칭 실패. 결과적으로 `pos` 가 일부 빠짐.

해결: `.*?` 와 `(.*?)` 로 충분히 lazy 하게, 그리고 column 명에 `` ` `` (backtick) 이 붙는 경우 (`` `value` ``) 도 처리.

**회고**: sqlmap 페이로드는 컬럼명을 backtick 으로 감싸기도 한다 (`` `user` ``, `` `value` ``). 정규식에 `` `? `` 옵션으로 처리. 한 번 testing 으로 모든 페이로드가 매칭되는지 확인해야 한다.

### 실패 4. `!=N` 한 줄만 보고 끝나는 줄 알았으나 일부 position 에 multiple `!=` 가 있었음

pos=10 같이 `!=49` 다음에 `>49`, `>51` 등이 더 등장. 처음에 "여러 `!=` 가 있으면 마지막을 쓸까 첫 번째를 쓸까" 헷갈렸다. 실험적으로 **첫 번째 `!=`** 가 정답 (sqlmap 은 한 번 확정하면 그 글자에 대해 더 비교 안 하므로, 그 이후의 `>` 는 다른 position 의 query 가 동일 패턴으로 매칭된 것).

**회고**: 정규식이 너무 광범위해서 다른 position 의 query 가 잘못 들어왔을 수 있음. 매 position 의 query 개수가 ~7-8 회 정도 (binary search of 7 bits) 가 정상. 그보다 많이 나오면 정규식이 다른 row 도 함께 매칭한 가능성.

이번 경우 다행히 첫 `!=` 가 모두 정답이라 결과는 맞았다. 다음번엔 정규식을 더 엄격히 (각 position 의 query 개수가 ~10 이내) 검증한 뒤 사용.
