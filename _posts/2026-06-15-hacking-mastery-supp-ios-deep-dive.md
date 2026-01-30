---
layout: post
title: "Network & App Hacking Mastery: Deep Dive - iOS Jailbreak Environment & Runtime Internals"
---

7주차에 "iOS는 안드로이드와 비슷하다"고 퉁치고 넘어갔지만, 사실 환경 구축부터 큰 장벽입니다.
진짜 iOS 해킹을 하려는 분들을 위해, 리얼 월드의 세팅과 내부 동작 원리를 파헤칩니다.

---

## ⛓️ The Lab Setup: Jailbreak

탈옥(Jailbreak) 없이는 Frida도 없습니다. (Non-jailbroken 방식도 있지만 제한적입니다.)

### 1. Checkra1n vs Palera1n vs Dopamine
*   **Checkra1n:** 하드웨어 취약점(BootROM)을 이용합니다. 아이폰 X 이하 기기만 가능하지만, **가장 안정적**이고 iOS 버전을 가리지 않습니다. 연구용 폰으로는 아이폰 7/8/X가 최고인 이유입니다.
*   **Palera1n / Dopamine:** 최신 iOS/기기를 지원하는 최신 탈옥툴입니다. "Rootless" 방식이라 파일 시스템 접근이 조금 다릅니다.

### 2. SSH & Frida Server
Cydia나 Sileo(앱스토어 같은 것)에서 `OpenSSH`를 설치합니다.
접속 비밀번호는 국룰인 `alpine`입니다. (꼭 바꾸세요!)
Frida Server도 Sileo에서 소스(`build.frida.re`)를 추가하고 설치하면 편합니다.

---

## 🍏 Objective-C Runtime Internals

Frida가 `ObjC.classes`로 메소드를 찾는 원리는 무엇일까요?
Objective-C의 모든 메소드 호출은 사실 C 함수 하나로 귀결됩니다.

### `objc_msgSend`

```objectivec
[user isPremium];
```
이 코드는 컴파일되면 아래처럼 바뀝니다.

```c
objc_msgSend(user, @selector(isPremium));
```

*   **receiver (`user`):** 메시지를 받을 객체
*   **selector (`isPremium`):** 메소드 이름 (ID)

런타임은 `user` 객체의 클래스 정보를 보고, `isPremium`이라는 이름표가 붙은 함수 포인터(IMP)를 찾아서 실행합니다.
Frida는 이 **IMP**를 바꿔치기(Swizzling)하는 것입니다.

그래서 `frida-trace -i "objc_msgSend"`를 하면 앱의 모든 동작이 로그로 찍히지만, 너무 많아서 폰이 멈출 수도 있습니다.

---

## 🔓 Decrypting IPAs (FairPlay)

앱스토어에서 받은 앱은 애플의 DRM인 **FairPlay**로 암호화되어 있습니다.
바이너리를 `IDA Pro`나 `Ghidra`에 넣으면 "Encrypted"라며 분석이 안 됩니다.
메모리에 로드된(복호화된) 상태를 덤프떠야 합니다.

### Tools
1.  **frida-ios-dump:** 가장 유명합니다. 스크립트 한 방이면 복호화된 `.ipa`가 나옵니다.
2.  **bagbak:** Node.js 기반의 덤프 툴입니다. 빠릅니다.

```bash
bagbak com.target.app
```
이렇게 얻은 깨끗한 바이너리를 IDA에 넣어야 비로소 분석이 시작됩니다.

---

## 🔌 CLI Power

아이폰을 USB로 꽂고 쓰는 유용한 명령어들입니다.

*   **`iproxy 2222 22`**: USB를 통해 SSH 터널을 뚫습니다. 이제 `ssh -p 2222 root@localhost`로 접속됩니다.
*   **`frida-ps -Uai`**: 설치된 모든 앱의 목록과 Bundle ID(`com.example.app`)를 보여줍니다.

iOS 해킹은 진입 장벽이 높지만, 그만큼 희소성이 있고 현상금(Bounty)도 셉니다.
오래된 아이폰 하나 구해서 시작해보세요.

**Break the jail.**
