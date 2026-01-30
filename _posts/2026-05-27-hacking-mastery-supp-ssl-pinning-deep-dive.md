---
layout: post
title: "Network & App Hacking Mastery: Deep Dive - SSL Pinning Under the Hood (Android Config, OkHttp & The Flutter Nightmare)"
---

4주차와 10주차에서 SSL Pinning의 개념과 우회법을 다뤘지만,
**"그래서 개발자들은 도대체 어떻게 Pinning을 구현하나요?"**라는 질문에 대한 답이 부족했습니다.
적을 알고 나를 알면 백전백승입니다. 구현 방식을 알면 우회 포인트가 보입니다.

오늘은 가장 대중적인 구현 방식 2가지와, 해커들이 가장 싫어하는 **Flutter** 앱의 Pinning을 다룹니다.

---

## 📌 Pinning Methods: 무엇을 고정하나?

1.  **Certificate Pinning (인증서 고정):** 서버의 인증서 파일 자체를 앱에 심어놓고 비교합니다.
    *   **단점:** 인증서는 유효기간이 있습니다. 만료돼서 갱신하면 앱도 업데이트해야 합니다. (비추천)
2.  **Public Key Pinning (공개키 고정):** 인증서 안에 있는 **공개키(Public Key)의 해시값(SHA-256)**을 고정합니다.
    *   **장점:** 인증서를 갱신해도 키 쌍(Key pair)을 그대로 쓰면 앱을 업데이트 안 해도 됩니다. 이것이 **SPKI (Subject Public Key Info) Pinning**이며, 표준입니다.

---

## 1️⃣ Implementation 1: Android Network Security Config

안드로이드 7.0(Nougat)부터 구글이 공식적으로 지원하는 방식입니다. 코드를 짤 필요도 없습니다.

### 개발자가 하는 일
`res/xml/network_security_config.xml` 파일을 만듭니다.

```xml
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set>
            <!-- 서버 공개키의 해시값 -->
            <pin digest="SHA-256">7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```
그리고 `AndroidManifest.xml`에서 이 파일을 참조합니다.

### 우리의 공략법 (Bypass)
1.  **Re-packaging:** `apktool`로 뜯어서 `AndroidManifest.xml`에서 `android:networkSecurityConfig` 속성을 지워버리면 끝입니다.
2.  **Frida:** 안드로이드 시스템 내부의 `NetworkSecurityTrustManager` 클래스를 후킹합니다.

---

## 2️⃣ Implementation 2: OkHttp (The De-facto Standard)

가장 많이 쓰이는 HTTP 클라이언트 라이브러리인 **OkHttp**를 쓰는 방식입니다.

### 개발자가 하는 일

```java
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("api.example.com", "sha256/7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=")
    .build();

OkHttpClient client = new OkHttpClient.Builder()
    .certificatePinner(pinner)
    .build();
```

### 우리의 공략법 (Bypass)
`okhttp3.CertificatePinner` 클래스의 `check(String hostname, List<Certificate> peerCertificates)` 메소드를 후킹합니다.
이 메소드가 예외(Exception)를 던지지 않고 조용히 리턴하게 만들면 됩니다.

---

## 😈 The Boss Fight: Flutter / Non-Standard Apps

가장 골치 아픈 녀석, **Flutter**입니다.

### The Problem
Flutter 앱은 Java(Dalvik/ART)를 쓰지 않습니다. **Dart** 언어로 작성되고, 자체적인 C++ 엔진(`libflutter.so`) 위에서 돌아갑니다.
*   시스템 프록시 설정을 무시합니다. (ProxyDroid 같은 걸 써야 함)
*   시스템 CA 저장소(TrustStore)를 안 씁니다. (자체적으로 내장함)
*   **Java.perform으로 아무리 후킹해도 소용없습니다.** (Java 레이어를 안 타니까요!)

### The Solution: Native Hooking
`libflutter.so` 내부의 SSL 검증 로직을 직접 후킹해야 합니다.

1.  **패턴 매칭:** `libflutter.so` 버전마다 오프셋이 다르므로, 바이너리 패턴으로 검증 함수(보통 `session_verify_cert_chain`이나 `ssl_crypto_x509_session_verify_cert_chain`)를 찾습니다.
    *   패턴 예시: `2d e9 f0 4f 85 b0 06 46 50 ...` (x86_64용)
2.  **도구:** `reflutter`라는 도구가 유명합니다.
    *   `reflutter target.apk`를 실행하면, 내부적으로 `libflutter.so`를 패치해서 트래픽을 가로챌 수 있게 개조된 APK를 만들어줍니다.
3.  **Frida Script:** `flutter-ssl-pinning-bypass` 스크립트들은 `Module.findBaseAddress("libflutter.so")` 후 패턴 스캔을 통해 검증 함수를 찾아 무력화합니다.

Flutter 앱을 분석할 때 mitmproxy에 아무것도 안 뜬다고 당황하지 마세요. 그것은 **Java의 세계가 아니기 때문**입니다.

**Dig deeper into the engine.**
