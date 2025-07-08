---
layout: post
title:  "Splunk 시리즈 Part 3: [Hands-on] Splunk Enterprise 설치 및 기본 데이터 수집"
date:   2025-07-08 10:00:00 +0900
categories: jekyll update
---

## 시작하며: 이론에서 실전으로

지난 두 파트에서는 Splunk의 개념과 대규모 환경을 위한 아키텍처 설계에 대해 알아보았습니다. 이제 탄탄한 이론적 배경을 바탕으로, 직접 Splunk를 설치하고 첫 번째 데이터를 수집해 볼 시간입니다.

이번 파트에서는 Splunk 아키텍처의 핵심인 **Splunk Enterprise(인덱서 및 서치 헤드 역할)**와 **Universal Forwarder(데이터 수집 역할)**를 직접 설치하고 연동하는 과정을 단계별로 안내합니다. 모든 과정을 따라오시면, 여러분의 로컬 서버 로그가 Splunk로 실시간으로 수집되는 것을 직접 확인할 수 있을 것입니다.

*참고: 본 가이드는 Rocky Linux 9 환경을 기준으로 작성되었으나, 다른 Linux 배포판에서도 거의 동일하게 적용할 수 있습니다. Windows 환경의 경우, 설치 과정이 GUI 기반으로 더 간단합니다.*

## Step 1: Splunk Enterprise 서버 설치하기

Splunk Enterprise는 데이터를 저장하고 검색하는 핵심 서버입니다. 테스트를 위해 모든 기능이 포함된 All-in-one 형태로 설치를 진행하겠습니다.

**1. Splunk Enterprise 다운로드**

먼저 Splunk 공식 홈페이지([splunk.com](https://www.splunk.com/))에 방문하여 회원가입 후, 무료 평가판(Free Trial)을 다운로드합니다. 본인 환경에 맞는 OS 버전을 선택합니다. 여기서는 Linux용 `.tgz` 파일을 다운로드하겠습니다.

```bash
# wget -O splunk-9.x.x-linux-2.6-x86_64.tgz '다운로드_링크'
wget -O splunk-9.2.1-85c59434436d-Linux-x86_64.tgz 'https://download.splunk.com/products/splunk/releases/9.2.1/linux/splunk-9.2.1-85c59434436d-Linux-x86_64.tgz'
```

**2. 압축 해제 및 설치**

다운로드한 파일을 `/opt` 디렉토리에 압축 해제합니다. Splunk는 별도의 설치 과정 없이, 압축을 해제하는 것만으로 설치가 완료됩니다.

```bash
# tar -xvzf splunk-9.2.1-....tgz -C /opt
```

**3. Splunk 실행**

이제 Splunk를 실행할 차례입니다. 처음 실행할 때는 라이선스 동의 및 관리자 계정 생성 과정이 필요합니다.

```bash
# /opt/splunk/bin/splunk start --accept-license
```

라이선스 동의 후, Splunk 관리자(admin) 계정의 비밀번호를 설정하라는 메시지가 나옵니다. 강력한 비밀번호를 설정하고 기억해두세요.

**4. Splunk 서비스 자동 실행 등록 (선택 사항)**

서버가 재부팅될 때마다 Splunk가 자동으로 시작되도록 서비스를 등록합니다.

```bash
# /opt/splunk/bin/splunk enable boot-start
```

**5. 방화벽 설정 및 접속 확인**

Splunk 웹 인터페이스는 기본적으로 8000번 포트를 사용합니다. 방화벽에서 8000번 포트를 열어줍니다.

```bash
# firewall-cmd --add-port=8000/tcp --permanent
# firewall-cmd --reload
```

이제 웹 브라우저에서 `http://<내_서버_IP>:8000`으로 접속하여, 위에서 생성한 admin 계정으로 로그인이 되는지 확인합니다. 멋진 Splunk 메인 화면이 여러분을 맞이할 것입니다.

## Step 2: 첫 데이터 수집하기 (2가지 방법)

이제 Splunk에 첫 데이터를 수집해 보겠습니다. 실제 서버 로그를 연동하기 전에, 누구나 동일한 환경에서 실습할 수 있도록 **샘플 데이터를 업로드하는 방법(권장)**과 실제 서버에 **Forwarder를 설치하여 수집하는 방법** 두 가지를 모두 안내합니다.

### 방법 1: 샘플 데이터 파일 업로드 (권장)

이 방법을 사용하면 별도의 서버나 Forwarder 설치 없이 즉시 다음 파트의 실습을 진행할 수 있습니다.

**1. 샘플 로그 파일 준비**

아래는 실습에 사용할 가상의 리눅스 보안 로그(`secure.log`) 샘플입니다. 이 내용을 복사하여 로컬 PC에 `secure.log` 라는 이름의 텍스트 파일로 저장하세요.

```log
Jul  8 10:00:11 security-server sshd[2512]: Failed password for invalid user admin from 112.220.11.11 port 22 ssh2
Jul  8 10:00:15 security-server sshd[2512]: Failed password for invalid user guest from 112.220.11.11 port 22 ssh2
Jul  8 10:00:20 security-server sshd[2512]: Failed password for root from 112.220.11.11 port 22 ssh2
Jul  8 10:01:05 security-server sshd[2518]: Accepted password for root from 192.168.1.10 port 5151 ssh2
Jul  8 10:02:30 security-server sshd[2525]: Failed password for root from 112.220.11.11 port 22 ssh2
Jul  8 10:05:10 security-server sshd[2530]: New session 123 for user splunk.
Jul  8 10:10:15 security-server sshd[2540]: Failed password for invalid user test from 211.45.12.33 port 22 ssh2
Jul  8 11:05:00 security-server sudo: splunk : TTY=pts/0 ; PWD=/home/splunk ; USER=root ; COMMAND=/bin/su
Jul  8 11:15:22 security-server sshd[3100]: Accepted password for splunk from 192.168.1.20 port 6231 ssh2
Jul  8 11:20:05 security-server sshd[3150]: Failed password for root from 211.45.12.33 port 22 ssh2
```

**2. Splunk 웹에서 파일 업로드**

1.  Splunk 웹 화면 상단의 **Settings > Add Data** 메뉴로 이동합니다.
2.  세 가지 옵션 중 **Upload** (Upload files from my computer)를 선택합니다.
3.  **Select File**을 클릭하여 방금 저장한 `secure.log` 파일을 선택합니다. 파일이 선택되면 **Next**를 클릭합니다.

**3. Source Type 및 Index 설정**

이제 Splunk에게 이 데이터가 어떤 종류이며, 어디에 저장할지 알려주어야 합니다.

1.  **Source Type:** **Select** 버튼을 누르고, 검색창에 `linux`를 입력한 뒤 나타나는 `linux_secure`를 선택합니다.
2.  **Index:** **Index** 설정에서 **Create a new index**를 선택합니다. **Index Name**에 `tutorial_logs` 라고 입력하고 **Save**를 클릭합니다.
3.  모든 설정이 완료되었으면 **Review** 버튼을, 마지막으로 **Submit** 버튼을 클릭하여 업로드를 완료합니다.

**4. 데이터 수집 확인**

"File has been uploaded successfully" 메시지가 나타나면 성공입니다. **Start Searching** 버튼을 클릭하면, `index="tutorial_logs" sourcetype="linux_secure"` 라는 검색어가 자동으로 입력된 검색 화면으로 이동하며, 방금 업로드한 10개의 로그 이벤트가 나타나는 것을 확인할 수 있습니다.

### 방법 2: Universal Forwarder로 직접 수집 (심화)

이 방법은 실제 운영 환경과 유사하게, 데이터를 생성하는 서버에 직접 Forwarder를 설치하여 데이터를 수집합니다.

**1. 데이터 수신 포트 설정**

먼저 Splunk Enterprise 서버가 Forwarder로부터 데이터를 받을 수 있도록 수신 포트(기본값: 9997)를 열어주어야 합니다.

1.  Splunk 웹 화면에서 **Settings > Forwarding and receiving** 메뉴로 이동합니다.
2.  **Receive data** 항목의 **Configure receiving**을 클릭합니다.
3.  **New Receiving Port**를 클릭하고, 포트 번호로 **9997**을 입력한 후 저장합니다.

**2. Universal Forwarder 설치 및 설정**

이제 데이터를 수집할 원격 서버(또는 Splunk Enterprise가 설치된 서버 자신)에 Universal Forwarder를 설치하고, Splunk Enterprise 서버를 바라보도록 설정합니다. (자세한 설치 과정은 Part 3의 초기 버전을 참고하세요.)

```bash
# 1. Forwarder 설치 후 실행 및 계정 생성
/opt/splunkforwarder/bin/splunk start --accept-license

# 2. 데이터 전송 대상(Splunk Enterprise 서버) 지정
/opt/splunkforwarder/bin/splunk add forward-server <Splunk_Enterprise_IP>:9997

# 3. 모니터링할 데이터 소스 추가 (예: /var/log/secure 파일)
/opt/splunkforwarder/bin/splunk add monitor /var/log/secure -sourcetype linux_secure -index os_logs

# 4. Forwarder 재시작
/opt/splunkforwarder/bin/splunk restart
```
*   `-sourcetype linux_secure`: 수집 시점에 Source Type을 `linux_secure`로 지정합니다.
*   `-index os_logs`: 데이터를 `os_logs` 라는 Index에 저장하도록 지정합니다.

**3. 데이터 수집 확인**

Splunk 검색창에서 `index="os_logs"` 를 검색하여 `/var/log/secure` 파일의 내용이 실시간으로 들어오는지 확인합니다.

## 다음 이야기

이번 파트에서는 Splunk에 데이터를 수집하는 두 가지 방법을 모두 배웠습니다. 특히 샘플 데이터를 활용하여 누구나 실습 환경을 꾸릴 수 있게 되었습니다. 하지만 아직 데이터는 정제되지 않은 '날 것'의 상태입니다.

다음 파트에서는 **"[Hands-on] 데이터의 가치를 높이는 첫걸음, 데이터 파싱과 인덱싱"**을 주제로, 수집된 데이터에 의미를 부여하고 검색 효율을 높이는 Source Type과 Index의 중요성에 대해 다시 한번 자세히(이번엔 개념 중심으로) 알아보겠습니다.

## 다음 이야기

이번 파트에서는 Splunk Enterprise와 Universal Forwarder를 직접 설치하고, 기본적인 데이터 수집 설정을 완료했습니다. 하지만 아직 데이터는 정제되지 않은 '날 것'의 상태입니다. 이 데이터를 어떻게 의미 있는 정보로 가공할 수 있을까요?

다음 파트에서는 **"[Hands-on] 데이터의 가치를 높이는 첫걸음, 데이터 파싱과 인덱싱"**을 주제로, 수집된 데이터에 의미를 부여하고 검색 효율을 높이는 Source Type과 Index의 중요성에 대해 알아보겠습니다.

---
