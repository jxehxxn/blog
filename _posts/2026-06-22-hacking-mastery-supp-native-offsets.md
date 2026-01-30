---
layout: post
title: "Network & App Hacking Mastery: Deep Dive - Advanced Native Hooking (Offsets & Memory Scanning)"
---

8주차에 `Module.findExportByName`으로 `open` 함수를 후킹했죠.
하지만 그건 "공개된(Exported)" 함수니까 쉬웠습니다.

개발자가 만든 **비공개 함수(Subroutine)**나, 심볼이 지워진(Stripped) 라이브러리는 어떻게 후킹할까요?
이름이 없으면 **주소(Offset)**로 찾아야 합니다.

---

## 📍 Strategy 1: Offset Hooking

### Step 1: Static Analysis (IDA/Ghidra)
`libtarget.so` 파일을 IDA로 엽니다.
분석을 통해 "아, 이 함수(`sub_12340`)가 비밀번호 검사 함수구나"라고 알아냅니다.
이때 함수의 시작 주소(Offset)가 `0x12340`이라는 것을 적어둡니다.

### Step 2: Calculate Absolute Address (ASLR)
메모리상의 실제 주소는 매번 바뀝니다. (**ASLR** 보안 기법)
`Base Address`(라이브러리가 로드된 시작 주소)에 `Offset`을 더해야 진짜 주소가 나옵니다.

### Step 3: Dynamic Hooking (Frida)

```javascript
// 1. 라이브러리의 시작 주소를 구한다.
var baseAddr = Module.findBaseAddress("libtarget.so");

if (baseAddr) {
    // 2. 오프셋을 더한다.
    var funcAddr = baseAddr.add(0x12340);
    
    // 3. 후킹!
    Interceptor.attach(funcAddr, {
        onEnter: function(args) {
            console.log("Secret function called!");
        }
    });
} else {
    console.log("Library not loaded yet.");
}
```

---

## 🔎 Strategy 2: Memory Scanning (Pattern Matching)

앱이 업데이트돼서 오프셋이 `0x12340`에서 `0x12380`으로 바뀌면요? 스크립트를 또 고쳐야 합니다.
귀찮죠? 그래서 **바이트 패턴(Signature)**을 씁니다.
함수의 기계어 코드(예: `55 48 89 E5 ...`)는 업데이트돼도 잘 안 바뀝니다.

```javascript
// 메모리 스캔
Memory.scan(baseAddr, 100000, "00 1A ?? FF", {
    onMatch: function(address, size) {
        console.log("Pattern found at: " + address);
        // 찾은 주소에 바로 후킹
        Interceptor.attach(address, { ... });
        
        return "stop"; // 하나 찾으면 중단
    },
    onError: function(reason) {
        console.log("Scan error: " + reason);
    }
});
```

---

## 🎣 Strategy 3: Hooking dlopen (Loading Time Hooking)

위의 코드들은 "라이브러리가 이미 로드된 상태"를 가정합니다.
앱이 켜지자마자 실행되는 함수라면? 우리가 스크립트를 주입하기도 전에 실행되고 끝날 수도 있습니다.

그래서 라이브러리를 불러오는 함수(`dlopen` 또는 `android_dlopen_ext`) 자체를 후킹합니다.

```javascript
var dlopen = Module.findExportByName(null, "android_dlopen_ext");

Interceptor.attach(dlopen, {
    onEnter: function(args) {
        this.path = Memory.readUtf8String(args[0]);
    },
    onLeave: function(retval) {
        if (this.path.indexOf("libtarget.so") > -1) {
            console.log("Library Loaded: " + this.path);
            // 이 시점부터 오프셋 후킹을 걸면 됩니다.
            hookSecretFunction(); 
        }
    }
});
```
이것이 바로 타이밍 싸움에서 이기는 법입니다.

**Find the hidden address.**
