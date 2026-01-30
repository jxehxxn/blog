---
layout: post
title: "Network & App Hacking Mastery: Deep Dive - The Art of APK Re-packaging & Smali Patching"
---

4주차 강의에서 우리는 "앱을 뜯어서 수정하는 방식(Re-packaging)은 구식이다"라고 했습니다.
하지만 **구식(Old School)이라고 해서 쓸모없는 것은 아닙니다.**

Frida는 강력하지만, 앱이 꺼지면 후킹도 풀립니다. 영구적인 변경(Permanent Patch)이 필요하거나, 강력한 안티 디버깅 때문에 Frida가 아예 붙지 못할 때는 이 방법을 써야 합니다.
이것이 바로 **안드로이드 리버싱의 기초**입니다.

---

## 🛠️ Tools of the Trade (준비물)

1.  **Apktool:** APK를 분해(Decompile)하고 다시 조립(Build)하는 도구입니다.
    *   다운로드: `ibotpeaches.github.io/Apktool`
2.  **Jadx-GUI:** 분해된 코드를 Java처럼 읽기 쉽게 보여주는 디컴파일러입니다.
    *   우리는 이걸로 **"어디를 고쳐야 할지"** 찾습니다.
3.  **Uber Apk Signer:** 조립된 APK에 서명(Sign)을 해주는 도구입니다. 서명이 없으면 안드로이드에 설치되지 않습니다.

---

## 🔬 The Process: Step-by-Step

### Step 1: Decompile (분해)
터미널에서 다음 명령어를 입력합니다.

```bash
apktool d target.apk
```
`target`이라는 폴더가 생깁니다. 내부 구조는 다음과 같습니다.
*   `AndroidManifest.xml`: 앱의 설정 파일
*   `res/`: 이미지, 레이아웃 등 리소스
*   `smali/`: **여기가 핵심입니다.** Java 바이트코드를 사람이 읽을 수 있게 변환한 코드입니다.

### Step 2: Analysis (분석)
`Jadx-GUI`로 `target.apk`를 엽니다.
SSL Pinning을 하는 코드를 찾습니다. 예를 들어 `CertificatePinner`를 검색해보니 `com.example.security.PinningManager` 클래스의 `check()` 메소드가 범인이라고 칩시다.

```java
// Java (Jadx가 보여주는 모습)
public void check() {
    if (!isTrusted) {
        throw new SSLHandshakeException("Untrusted!");
    }
}
```

### Step 3: Editing Smali (수술 집도)
이제 `smali` 폴더로 가서 `com/example/security/PinningManager.smali` 파일을 엽니다.
Java와는 다르게 생겼습니다. 이것이 **Smali** 언어입니다.

```smali
# Smali (Before)
.method public check()V
    .locals 2
    # ... (검사 로직) ...
    if-eqz v0, :cond_0
    
    # 예외를 던지는 부분
    new-instance v1, Ljavax/net/ssl/SSLHandshakeException;
    invoke-direct {v1}, Ljavax/net/ssl/SSLHandshakeException;-><init>()V
    throw v1

    :cond_0
    return-void
.end method
```

우리는 이 검사 로직을 다 날리고, 그냥 아무 일도 없이 리턴하게 만들 겁니다.

```smali
# Smali (After)
.method public check()V
    .locals 0
    
    # 쿨하게 바로 리턴
    return-void
.end method
```
코드가 훨씬 짧아졌죠? "검사고 뭐고 난 모르겠고 그냥 끝내"라는 뜻입니다.

### Step 4: Rebuild (재조립)
수정한 내용을 다시 APK로 만듭니다.

```bash
apktool b target -o modified.apk
```

### Step 5: Sign (서명)
재조립된 APK는 서명이 깨져있어 설치가 안 됩니다. 다시 서명합니다.

```bash
java -jar uber-apk-signer.jar -a modified.apk
```
이제 `modified-aligned-debugSigned.apk`가 생성됩니다. 이걸 폰에 설치하면 됩니다.

---

## 🧩 Dealing with Split APKs (XAPK/APKS)

요즘 구글 플레이 스토어는 **App Bundle**을 써서 APK가 여러 개로 쪼개져 있습니다. (base.apk, config.arm64.apk 등)
이 경우 `Apktool`로 그냥 풀면 에러가 납니다.

1.  **Anti-Split 도구**를 써서 하나로 합치거나,
2.  가장 핵심인 `base.apk`만 패치하고, 설치할 때 모든 APK를 같이 설치(`adb install-multiple`)해야 합니다.

이 방법은 Frida보다 귀찮고 복잡하지만, **앱 자체를 영구적으로 변조**할 수 있다는 점에서 해커에게는 강력한 무기입니다.

**Unpack, Patch, Repack.**
