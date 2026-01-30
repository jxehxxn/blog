---
layout: post
title: "Network & App Hacking Mastery: Deep Dive - Frida Tracing Masterclass (frida-trace & Stalker)"
---

5주차에 우리는 `Interceptor.attach`로 함수 하나를 후킹하는 법을 배웠습니다.
하지만 앱이 무슨 함수를 호출하는지 모른다면요? 수천 개의 함수를 일일이 찍어볼 순 없습니다.
이럴 때 필요한 것이 **광역 추적(Mass tracing)**입니다.

---

## 🕵️ Tool 1: frida-trace (The Auto-Hooker)

Frida를 설치하면 딸려오는 `frida-trace`는 마법의 도구입니다.
와일드카드(`*`)를 써서 수백 개의 함수를 동시에 후킹합니다.

### 1. Java 메소드 추적
"이 앱이 암호화(`Crypto`) 관련된 무슨 클래스를 쓰는지 궁금해."

```bash
frida-trace -U -f com.target.app -j "*!*Crypto*"
```
*   `-U`: USB 연결
*   `-f`: 앱 실행 (Spawn)
*   `-j`: Java 클래스 필터. `*!*Crypto*`는 "모든 패키지(!)의 이름에 Crypto가 들어가는 모든 메소드"를 의미합니다.

### 2. Native 함수 추적
"이 앱이 어떤 파일을 여는지(`open`) 궁금해."

```bash
frida-trace -U -f com.target.app -i "open"
```
*   `-i`: Native Export 함수 필터.

### 3. Handlers (자동 생성된 스크립트)
`frida-trace`를 실행하면 현재 폴더에 `__handlers__` 폴더가 생깁니다.
그 안에 각 함수별 `.js` 파일이 자동 생성됩니다.
`open.js`를 열어서 `onEnter`를 수정하면, 인자값을 더 예쁘게 출력할 수 있습니다.

```javascript
onEnter: function (log, args, state) {
    // 자동 생성된 코드: log("open(" + args[0] + ")");
    // 수정: 문자열로 읽어서 출력
    log("open(" + Memory.readUtf8String(args[0]) + ")");
}
```

---

## 👣 Tool 2: Frida Stalker (The Instruction Tracer)

만약 함수 이름이 다 지워졌거나(Obfuscation), 함수 단위가 아니라 **CPU 명령어 단위**로 추적하고 싶다면요?
암호화 알고리즘이 숨겨져 있을 때, 어떤 연산이 일어나는지 보고 싶다면요?

**Stalker**는 Frida의 심장과도 같은 엔진으로, 스레드의 실행 흐름을 가로채서 명령어를 하나하나 뜯어볼 수 있습니다.

### Code Example

```javascript
var mainThread = Process.enumerateThreads()[0];

Stalker.follow(mainThread.id, {
    events: {
        call: true, // 함수 호출(CALL) 명령어만 추적
        ret: false,
        exec: false // 모든 명령어 추적 (너무 느려짐 주의!)
    },
    onReceive: function (events) {
        var logs = Stalker.parse(events);
        console.log(logs);
    }
});
```
이걸 실행하면 엄청난 양의 로그가 쏟아집니다.
그래서 보통은 특정 주소 범위(특정 라이브러리 내부)만 추적하도록 필터를 겁니다.

---

## 🧠 Advanced: Memory Tracing (Watchpoints)

"이 변수(`secret_key`)를 누가 읽거나 쓰는지 알고 싶어."
디버거의 **하드웨어 브레이크포인트** 같은 기능입니다.

```javascript
// 감시할 메모리 주소와 크기
var targetAddr = ptr("0x12345678");
var targetSize = 4;

MemoryAccessMonitor.enable({
    base: targetAddr,
    size: targetSize
}, {
    onAccess: function (details) {
        console.log("Memory Accessed!");
        console.log("Operation: " + details.operation); // read or write
        console.log("From IP: " + details.from); // 누가 접근했는지(Instruction Pointer)
    }
});
```

`frida-trace`로 숲을 보고, `Stalker`로 나무를 보고, `MemoryAccessMonitor`로 나뭇잎을 봅니다.
이 세 가지를 자유자재로 다룬다면 분석하지 못할 앱은 없습니다.

**Trace everything.**
