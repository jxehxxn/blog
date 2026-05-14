---
layout: post
title:  "Dreamhack 워게임 baby-Case 풀이 — Nginx와 Express의 대소문자 케이스 어긋남"
date:   2026-05-14 21:30:00 +0900
categories: security web wargame dreamhack writeup
---

## 들어가며

이번 글에서는 Dreamhack 웹해킹 워게임 **baby-Case** (난이도 Bronze 3) 문제를 처음 접하는 정보보안 입문자의 시점에서 단계별로 풀어봅니다. 문제 제목 그대로 "Case(대소문자)"가 핵심이며, 코드는 짧지만 **프록시(Nginx)와 백엔드(Express) 사이에서 라우팅 규칙이 서로 다른 경우** 어떤 보안 이슈가 생기는지를 보여주는 아주 교과서적인 예제입니다.

문제 설명은 단 한 줄, **"Bypass 👶filter"** 뿐입니다. 어떤 필터인지조차 알려주지 않으므로 직접 소스 코드를 읽고 추론해야 합니다.

> Spoiler: 최종 페이로드는 `POST /Shop` 한 줄짜리 `curl` 명령으로 끝납니다. 하지만 그 한 줄에 도달하기까지의 사고 과정이 이 문제의 진짜 가치입니다.

## 1. 문제 파일 들여다보기

문제 페이지에서 "문제 파일 받기"를 눌러 받은 ZIP을 풀면 아래와 같은 구조가 나옵니다.

```
baby-Case/
├── docker-compose.yml
└── deploy/
    ├── app/
    │   ├── app.js
    │   ├── ag.js
    │   ├── Dockerfile
    │   ├── package.json
    │   └── package-lock.json
    └── nginx/
        └── nginx.conf
```

처음 보고 가장 먼저 떠올려야 하는 사실 두 가지가 있습니다.

1. `docker-compose.yml`이 있다 → 단일 서비스가 아니라 **여러 컨테이너가 협력**한다.
2. `nginx/`와 `app/`이 따로 있다 → **앞단(Nginx)과 뒷단(Express) 사이의 경계**가 존재한다.

웹해킹에서 "두 개 이상의 컴포넌트가 같은 입력을 서로 다르게 해석한다"는 사실은 거의 항상 취약점의 시그널입니다. **HTTP request smuggling**, **path traversal via normalization mismatch**, **case-sensitivity mismatch** 같은 클래스가 모두 이 카테고리에 속합니다.

### 1-1. `docker-compose.yml`

```yaml
version: "3.9"
services:
  nginx:
    image: nginx:1.22.0
    ports:
      - "8000:80"
    volumes:
      - ./deploy/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
  app:
    build:
      context: ./deploy/app
```

외부에 노출된 포트는 `8000` 하나이며, 그 트래픽은 모두 **Nginx**가 먼저 받습니다. 사용자는 절대 `app` 컨테이너에 직접 닿을 수 없고, 반드시 Nginx를 한 번 거쳐야 합니다.

### 1-2. `nginx/nginx.conf` — 첫 번째 필터

```nginx
events {
    worker_connections  1024;
}

http {
    server {
        listen 80;
        listen [::]:80;
        server_name  _;

        location = /shop {
            deny all;
        }

        location = /shop/ {
            deny all;
        }

        location / {
            proxy_pass http://app:3000/;
        }
    }
}
```

이 설정에서 보안적으로 흥미로운 부분은 `location =`입니다. Nginx의 `location` 지시어는 **수식어(modifier)**에 따라 매칭 방식이 달라집니다.

- `location /shop` → **prefix(접두) 매칭**. `/shop`으로 시작하는 모든 경로(`/shop`, `/shop/anything`, `/shopABC`).
- `location = /shop` → **exact(정확) 매칭**. 오직 `/shop` 하나만.
- `location ~ /shop` → 대소문자 구분 정규식.
- `location ~* /shop` → 대소문자 무시 정규식.

여기서는 `=`이 쓰였습니다. 따라서 차단되는 경로는 **정확히 `/shop`와 정확히 `/shop/`**, 단 두 개뿐입니다. 게다가 Nginx의 location 매칭은 **기본적으로 대소문자를 구분**합니다(`~*`나 OS의 case-insensitive filesystem이 아닌 한). 즉, `/Shop`이나 `/SHOP`은 `/shop`과 **다른 경로**로 취급됩니다.

> 💡 비유로 설명하면, 호텔 보안요원이 "이름 적힌 명단에 `john` 있는 사람은 들여보내지 마"라고 들었는데, `John`이라고 적힌 신분증을 가진 사람은 그냥 통과시키는 상황입니다. 명단에 적힌 글자와 1바이트라도 다르면 차단 로직이 발동하지 않습니다.

### 1-3. `app/app.js` — 두 번째 필터

```javascript
const express = require("express")
const words = require("./ag")

const app = express()
const PORT = 3000
app.use(express.urlencoded({ extended: true }))

function search(words, leg) {
    return words.find(word => word.name === leg.toUpperCase())
}

app.get("/", (req, res) => {
    return res.send("hi guest")
})

app.post("/shop", (req, res) => {
    const leg = req.body.leg

    if (leg == 'FLAG') {
        return res.status(403).send("Access Denied")
    }

    const obj = search(words, leg)

    if (obj) {
        return res.send(JSON.stringify(obj))
    }

    return res.status(404).send("Nothing")
})

app.listen(PORT, () => {
    console.log(`[+] Started on ${PORT}`)
})
```

여기서 가장 눈에 띄는 코드는 이 세 줄입니다.

1. `app.post("/shop", ...)` — Express 라우팅
2. `if (leg == 'FLAG') return 403` — 두 번째 필터
3. `word.name === leg.toUpperCase()` — 실제 검색 로직

`search`가 `leg.toUpperCase()`를 쓰고 있다는 것은 이 함수가 **대소문자 무관하게 매칭하기를 원한다는 의도**를 보여줍니다. 그런데 그 위의 차단 조건은 `leg == 'FLAG'` (대소문자를 구분하는 비교)뿐입니다. 의도와 구현이 **어긋나** 있죠.

### 1-4. `app/ag.js` — 데이터셋

```javascript
module.exports = [
    { "id": 1, "name": "FLAG", "description": "DH{fake_flag}" },
    { "id": 2, "name": "DRAG", "description": "..." },
    { "id": 3, "name": "SLAG", "description": "..." },
    { "id": 4, "name": "SWAG", "description": "..." }
]
```

배포 환경에서는 1번 항목의 `description`이 진짜 플래그로 덮어쓰여 있을 것입니다. 우리의 목표는 **1번 객체의 `description` 필드를 응답으로 받아내는 것**입니다.

## 2. 표면 정찰

서버 부팅 후 가장 먼저 한 일은 단순히 `/`에 GET 요청을 보내본 것입니다.

```bash
$ curl -sv http://host8.dreamhack.games:22660/
< HTTP/1.1 200 OK
< Server: nginx/1.22.0
< X-Powered-By: Express
hi guest
```

응답 헤더에서 두 가지 단서를 얻습니다.

- `Server: nginx/1.22.0` — 앞단은 Nginx 1.22.0
- `X-Powered-By: Express` — 뒷단은 Express

이는 우리가 분석한 `docker-compose.yml`과 정확히 일치하고, **소스 코드가 실제 배포본과 같다**는 확신을 줍니다.

다음으로 차단 동작을 확인해봅니다.

```bash
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://host8.dreamhack.games:22660/shop -d "leg=DRAG"
403
```

정직하게 `/shop`을 두드리면 Nginx가 403을 돌려줍니다. 이 응답은 **Express가 아니라 Nginx**가 만든 것임에 주의하세요. (Express라면 `Access Denied` 텍스트나 다른 응답을 줬을 것입니다.)

## 3. 취약점 모델링

이 시점에서 머릿속에 그려야 하는 그림은 다음과 같습니다.

```
[클라이언트]
    │  POST /??? leg=???
    ▼
┌──────────────┐
│ Nginx 1.22.0 │   ← location = /shop, /shop/ 만 차단
└──────┬───────┘
       │ proxy_pass http://app:3000/   (그 외 모든 경로는 통과)
       ▼
┌──────────────┐
│  Express 4   │   ← 라우팅: 기본 case-INsensitive
└──────────────┘
```

- Nginx는 **정확히 `/shop`만 차단**한다.
- Express는 기본적으로 **라우팅 시 대소문자를 무시**한다 (`app.set('case sensitive routing', true)`가 호출되지 않았음).
- 따라서 `/Shop`, `/SHOP`, `/sHoP` 같은 경로는 **Nginx의 차단을 우회하면서도 Express는 정상적으로 `/shop` 핸들러를 호출**한다.

이것이 첫 번째 우회입니다. "필터"의 정체죠.

두 번째 우회는 앱 내부에 있습니다.

```javascript
if (leg == 'FLAG') return res.status(403).send("Access Denied")
// ...
return words.find(word => word.name === leg.toUpperCase())
```

- `leg == 'FLAG'`는 **두 문자열이 정확히 일치**할 때만 참입니다. JS의 `==`는 양쪽이 모두 문자열일 때 `===`와 동일하게 동작합니다.
- 반면 `leg.toUpperCase()`는 **소문자도 대문자로 만든 뒤 비교**합니다.

`leg = "flag"`(소문자)를 보내면:

- `"flag" == "FLAG"` → **false** (검사 통과)
- `"flag".toUpperCase()` → `"FLAG"` → `words[0]`에 매칭 → 플래그 반환

> 💡 이 두 우회는 모두 **"같은 의미인데 표기만 다른 입력"** 을 활용합니다. 보안 점검에서 항상 자문해야 하는 질문: "내가 막고 싶은 값과 동치인 표현이 또 있나? 인코딩, 대소문자, 공백, 트레일링 슬래시, 유니코드 정규화..."

## 4. 익스플로잇

페이로드는 두 가지를 동시에 만족해야 합니다.

1. 경로: `/shop` 이외의 표기 → `/Shop` 선택
2. 본문: `leg`가 `FLAG`와 정확히 같지 않으면서 `toUpperCase()` 결과가 `FLAG`인 값 → `flag`

```bash
$ curl -s -X POST http://host8.dreamhack.games:22660/Shop -d "leg=flag"
{"id":1,"name":"FLAG","description":"DH{167e709c31556e37207e46db78f34ab1d6919e0789e3441b5740f02143169ecc}"}
```

플래그가 그대로 응답에 떨어집니다. `DH{167e709c...169ecc}`.

비교를 위해 차단되는 케이스도 함께 확인해두면 학습에 도움이 됩니다.

```bash
# (1) 소문자 + 정확한 /shop → Nginx에서 차단
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST .../shop -d "leg=flag"
403   # Nginx

# (2) /Shop + leg=FLAG (대문자) → Express의 조건문에 걸림
$ curl -s -X POST .../Shop -d "leg=FLAG"
Access Denied   # 403 from Express

# (3) /Shop + leg=flag (소문자) → 성공
$ curl -s -X POST .../Shop -d "leg=flag"
{"id":1,"name":"FLAG","description":"DH{...}"}
```

세 케이스를 모두 시도해보면, 각 필터가 어디서 작동하고 어디서 무너지는지 손에 잡힙니다.

## 5. 취약점 해설 — Parser/Path Confusion

이 문제는 통상 **Parser Differential** 또는 **Path Confusion** 클래스로 분류됩니다.

> 같은 요청을 두 개의 컴포넌트가 서로 다르게 해석할 때, 한쪽의 보안 정책이 다른 쪽에서 의미를 잃는 현상.

대표적인 실사례:

- **CVE-2022-24348 (Argo CD)**: URL 정규화 차이로 RBAC 우회.
- **PortSwigger의 HTTP Path Manipulation 시리즈**: 같은 경로를 reverse proxy와 origin이 다르게 정규화.
- **Cloudflare/Nginx + Node.js**: trailing dot, 백슬래시, 대소문자, 퍼센트 인코딩(`%2e`, `%2f`) 등으로 캐시 키와 권한 검사 우회.

`location = /shop`처럼 **"화이트리스트가 아닌 블랙리스트"**, 그리고 **정규화 없는 정확 매칭**은 거의 항상 우회 가능성을 만듭니다.

### 5-1. 위험성

- 인증·인가 우회: 관리자 페이지(`/admin`)를 같은 식으로 막아두면 `/Admin`으로 그대로 노출됩니다.
- WAF/CDN 우회: CDN이 캐시한 정책과 오리진의 라우팅이 어긋나면 캐시 포이즈닝이나 인증 우회로 이어집니다.
- 로깅 누락: 보안팀이 `/shop` 접근만 모니터링한다면, `/Shop` 트래픽은 시야에서 사라집니다.

### 5-2. 올바른 방어

문제의 의도대로라면 두 가지 모두 고쳐야 합니다.

1. Nginx에서 **대소문자 무시 정규식**을 쓰거나, 더 나은 방법은 **앱 단에서 인증·인가를 처리하고 프록시는 단순 전달**만 시키는 것입니다.

   ```nginx
   location ~* ^/shop/?$ {
       deny all;
   }
   ```

2. Express에서 동등성 비교를 정규화 후에 수행합니다.

   ```javascript
   if (String(leg).toUpperCase() === 'FLAG') {
       return res.status(403).send("Access Denied")
   }
   ```

   더 나아가, `leg`가 문자열인지 타입 체크를 추가해 prototype pollution이나 array confusion(`leg=foo&leg=bar` → `["foo","bar"]`) 같은 추가 케이스도 방어할 수 있습니다.

## 6. 정리 — 입문자가 가져갈 교훈

- 소스 코드를 받았다면 **컴포넌트의 경계**를 먼저 그려라. 어떤 입력이 어떤 순서로 어떤 컴포넌트를 통과하는지 명확히 한 다음에야 "어디가 약한지"가 보인다.
- 블랙리스트는 의심하라. 특히 **정확 일치/접두 일치**, **대소문자 구분**, **인코딩 미정규화** 같은 비교 방식은 항상 같은 의미의 다른 표기를 시도해볼 가치가 있다.
- JS의 `==`, `===`, `.toUpperCase()`, `.toLowerCase()`, `String()` 등 **자료형 변환과 정규화가 끼어드는 모든 지점**이 잠재적 우회 후보다.
- 직접 손으로 페이로드를 만들어 차단되는 케이스(`/shop`)와 통과하는 케이스(`/Shop`)를 **둘 다** 관찰하면 보안 메커니즘이 어떻게 작동하는지 직관이 생긴다.

문제 풀이 자체는 5분도 안 걸리지만, 거기서 끄집어낼 수 있는 개념(reverse proxy 라우팅, parser differential, 정규화 우회)은 실무에서 그대로 쓰입니다. 같은 카테고리의 더 어려운 도전 과제로 Dreamhack의 `BypassIF`나 `Path Traversal` 류의 문제도 함께 풀어보면 좋겠습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

문제를 푸는 과정에서 곧바로 정답에 도달한 것은 아닙니다. 같은 실수를 다시 하지 않기 위해 기록합니다.

### 실패 1. 이전 세션의 ZIP을 그대로 사용했다

`~/Downloads/`에는 같은 문제(혹은 `id=1401` 외 다른 문제)의 ZIP이 이미 남아 있었고, 첫 분석을 그 캐시된 파일로 진행했습니다. 그 파일은 Python Flask + Selenium 봇을 사용하는 **DOM XSS 류 문제**(CSP `strict-dynamic` 우회를 요구하는 다른 챌린지)였습니다. 한참 동안 "CSP를 어떻게 우회할까", "innerHTML 기반의 가젯이 뭘까"를 고민하다가, `curl /`이 `hi guest`를 돌려주면서 `X-Powered-By: Express` 헤더를 보고서야 "지금 실행 중인 서버는 Node.js이고 내가 보고 있는 코드는 Flask다"라는 모순을 알아챘습니다.

**회고**: 분석 시작 전에 반드시 한 가지를 확인해야 합니다. **소스 코드의 framework/런타임이 실제 서버 응답 헤더(`Server`, `X-Powered-By`)와 일치하는가?** 두 정보가 어긋나면, 즉시 소스 파일을 다시 받아야 합니다. 다음부터는 분석에 들어가기 전 다음 체크리스트를 먼저 수행하겠습니다.

1. ZIP 파일명(UUID)을 메모하고 다운로드 직전/직후 디렉터리 스냅샷을 비교한다.
2. 첫 1분 내 `curl -I`로 서버 헤더를 확인하고, 소스의 Dockerfile/언어와 일치하는지 검증한다.
3. 일치하지 않으면 곧바로 ZIP을 다시 받는다 (캐시된 파일을 의심한다).

### 실패 2. VM 부팅 직후 너무 빨리 요청을 보냈다

서버가 부팅되기 전 잠깐 동안 Dreamhack의 인프라가 "기본 응답"으로 빈 페이지나 다른 응답을 줄 가능성이 있었습니다(실제로는 같은 `hi guest`였지만, 만약 부팅 중이었다면 다른 응답이 와서 분석을 혼란시켰을 것입니다).

**회고**: VM 상태 API(`GET /api/v1/wargame/challenges/{id}/live/`)의 `state` 값이 `Running`인지 먼저 확인한 뒤 curl을 보낸다.

### 실패 3. 브라우저 자동화 중 "구독 권유" 모달을 인지하지 못함

서버 생성 버튼을 클릭하기 위한 좌표 클릭이 모달에 가려져 한동안 동작하지 않았습니다.

**회고**: 클릭 전 스크린샷으로 모달/오버레이가 떠 있지 않은지 확인하고, 가능한 경우 좌표 클릭 대신 `ref_id`/`selector` 기반 클릭을 사용한다. 모달이 보이면 먼저 닫는다.

이 세 가지가 다음 문제를 풀 때 시간을 가장 많이 줄여줄 학습 포인트입니다.
