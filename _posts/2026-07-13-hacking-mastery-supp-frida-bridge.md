---
layout: post
title: "Network & App Hacking Mastery: Deep Dive - Building the Ultimate Bridge (Frida to Mitmproxy)"
---

11주차에 "Frida와 mitmproxy를 연결해서 암호화를 뚫는다"는 개념을 설명했습니다.
많은 분들이 **"그래서 코드는요?"**라고 물어보셨습니다.
여기, 그 **Ultimate Bridge**의 전체 소스 코드를 공개합니다.

---

## 🏗️ Architecture

1.  **App (Frida):** 암호화 직전 평문 가로채기 -> PC로 전송.
2.  **Bridge Server (Python):** PC에서 대기하다가 데이터를 받아서 mitmproxy로 토스.
3.  **Mitmproxy:** 평문을 보여주고, 수정하면 다시 Bridge로 토스.
4.  **App (Frida):** 수정된 평문을 받아서 원래 암호화 함수에 집어넣음.

---

## 🌉 Component 1: The Bridge Server (Python)

Flask로 간단한 웹 서버를 만듭니다. Frida가 여기로 POST 요청을 보낼 겁니다.

```python
# bridge.py
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)
PROXY_URL = "http://127.0.0.1:8080" # mitmproxy 주소

@app.route('/encrypt', methods=['POST'])
def encrypt_hook():
    data = request.json['payload']
    print(f"[*] Received from App: {data}")
    
    # mitmproxy를 거치게 하기 위해 프록시를 태워서 요청을 보냄
    # (여기서 mitmproxy가 데이터를 잡아서 수정할 수 있음!)
    try:
        response = requests.post("http://echo.it", json=data, proxies={"http": PROXY_URL})
        modified_data = response.json() # mitmproxy가 수정한 데이터
    except:
        modified_data = data

    return jsonify({"payload": modified_data})

if __name__ == '__main__':
    app.run(port=5000)
```

---

## 💉 Component 2: The Frida Script

앱 내부에서 `AES.encrypt`를 후킹하고, Bridge 서버와 통신합니다.

```javascript
// hook.js
var AES = Java.use("com.example.security.AES");

AES.encrypt.implementation = function(plaintext) {
    console.log("[Frida] Original: " + plaintext);
    
    var modifiedPlaintext = plaintext;

    // Bridge Server로 전송 (동기식 요청 필요 - 여기선 개념적으로 설명)
    // 실제로는 Java의 HttpURLConnection을 사용하여 127.0.0.1:5000/encrypt 로 보냄
    
    var url = Java.use("java.net.URL").$new("http://127.0.0.1:5000/encrypt");
    var conn = url.openConnection();
    conn.setRequestMethod("POST");
    conn.setDoOutput(true);
    // ... (JSON 데이터 전송 및 응답 수신 코드 생략) ...
    
    // Bridge가 준(mitmproxy가 수정한) 데이터로 교체
    // modifiedPlaintext = responseFromBridge;
    
    console.log("[Frida] Modified: " + modifiedPlaintext);
    
    // 수정된 데이터로 암호화 수행
    return this.encrypt(modifiedPlaintext);
};
```

---

## 🚦 Component 3: The Mitmproxy Setup

이제 `mitmproxy`를 켜고 `http://echo.it`으로 가는 요청을 잡으면 됩니다.
그 요청의 Body에는 **평문(Plaintext)**이 들어있습니다.

1.  앱에서 "로그인" 클릭.
2.  Frida가 평문 ID/PW를 Bridge로 보냄.
3.  Bridge가 mitmproxy로 보냄.
4.  **mitmproxy 화면에 ID/PW가 평문으로 뜸!**
5.  여러분이 ID를 `admin`으로 수정.
6.  Bridge가 받아서 Frida로 리턴.
7.  앱은 `admin`을 암호화해서 서버로 전송.

이것이 바로 **End-to-End Encryption을 무력화하는 완벽한 파이프라인**입니다.

**Bridge the gap.**
