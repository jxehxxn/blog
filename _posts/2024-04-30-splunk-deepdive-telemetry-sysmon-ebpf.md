---
layout: post
title: "고급 엔드포인트 및 클라우드 네이티브 텔레메트리 분석 (Sysmon, auditd, Falco, eBPF)"
date: 2024-04-30 09:00:00 +0900
categories: [Security, Splunk]
tags: [Splunk, Sysmon, auditd, Falco, eBPF, Telemetry]
---

보안 업계에서 30년 가까이 구르며 깨달은 진리가 하나 있습니다. "로그가 없으면 사건도 없다"는 겁니다. 하지만 단순히 로그가 있는 것만으로는 부족합니다. 공격자는 점점 더 교묘해지고, 표준 OS 로그만으로는 그들의 흔적을 찾기 어렵습니다. 오늘은 윈도우, 리눅스, 그리고 클라우드 네이티브 환경에서 어떻게 깊이 있는 텔레메트리를 확보하고 이를 Splunk로 분석하는지 3시간 분량의 강의 수준으로 깊게 파헤쳐 보겠습니다.

## 1. 왜 표준 OS 로그만으로는 부족한가?

윈도우 이벤트 로그나 리눅스의 기본 syslog는 운영 체제의 상태를 기록하기 위해 만들어졌지, 보안 침해 사고를 추적하기 위해 설계된 게 아닙니다.

예를 들어, 윈도우의 기본 프로세스 생성 로그(Event ID 4688)는 명령줄 인수를 기록하지 않는 경우가 많습니다. 공격자가 `powershell.exe -ExecutionPolicy Bypass -EncodedCommand ...` 같은 명령을 실행했을 때, 단순히 파워쉘이 실행되었다는 사실만 알아서는 아무것도 할 수 없습니다.

리눅스도 마찬가지입니다. 누가 파일을 수정했는지, 어떤 프로세스가 네트워크 연결을 시도했는지 기본 로그만으로는 파악하기 힘듭니다. 특히 컨테이너 환경으로 넘어가면 문제는 더 심각해집니다. 컨테이너는 금방 사라지고, 그 안에서 벌어지는 일들은 호스트 OS의 기본 로그에 잘 남지 않기 때문입니다.

그래서 우리는 **깊은 텔레메트리(Deep Telemetry)**가 필요합니다.

---

## 2. Windows Sysmon: 엔드포인트의 눈

Sysmon(System Monitor)은 마이크로소프트 Sysinternals 패키지의 일부로, 시스템 서비스 및 장치 드라이버로 설치되어 재부팅 후에도 시스템 활동을 모니터링하고 기록합니다.

### 주요 이벤트 ID (Event IDs)

우리가 주목해야 할 핵심 이벤트는 다음과 같습니다.

*   **Event ID 1: Process Creation**: 프로세스가 시작될 때 기록됩니다. 부모 프로세스, 명령줄 인수, 파일 해시 등을 포함합니다.
*   **Event ID 3: Network Connection**: TCP/UDP 연결을 기록합니다. 어떤 프로세스가 어디로 연결을 시도했는지 보여줍니다.
*   **Event ID 11: FileCreate**: 파일이 생성되거나 덮어씌워질 때 기록됩니다. 랜섬웨어 탐지에 필수적입니다.

### Sysmon 설치 및 설정

Sysmon은 XML 설정 파일을 통해 어떤 이벤트를 수집할지 결정합니다. SwiftOnSecurity의 설정을 참고하는 게 업계 표준입니다.

```xml
<Sysmon schemaversion="4.30">
  <HashAlgorithms>md5,sha256</HashAlgorithms>
  <EventFiltering>
    <!-- Process Create -->
    <RuleGroup name="" groupRelation="or">
      <ProcessCreate onmatch="exclude">
        <Image condition="end with">chrome.exe</Image>
      </ProcessCreate>
      <ProcessCreate onmatch="include">
        <Image condition="begin with">C:\Windows\System32\WindowsPowerShell</Image>
        <ParentImage condition="image">cmd.exe</ParentImage>
      </ProcessCreate>
    </RuleGroup>

    <!-- File Create -->
    <RuleGroup name="" groupRelation="or">
      <FileCreate onmatch="include">
        <TargetFilename condition="contains">\Downloads\</TargetFilename>
        <Extension condition="is">.exe</Extension>
      </FileCreate>
    </RuleGroup>
  </EventFiltering>
</Sysmon>
```

설치 명령은 다음과 같습니다.
`sysmon.exe -i config.xml -accepteula`

### Splunk로 데이터 보내기 (Universal Forwarder)

Splunk UF의 `inputs.conf`에 다음 설정을 추가합니다.

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
renderXml = 1
index = endpoint_logs
sourcetype = xmlwineventlog
```

---

## 3. Linux auditd: 커널 수준의 감시

리눅스 환경에서는 `auditd`를 사용해 시스템 호출(Syscall)을 모니터링합니다. 파일 접근, 권한 변경, 네트워크 소켓 생성 등을 추적할 수 있습니다.

### audit.rules 설정

`/etc/audit/rules.d/audit.rules` 파일에 규칙을 추가합니다.

```bash
# 모든 실행 파일 실행 추적
-a always,exit -F arch=b64 -S execve -k proc_exec

# /etc/shadow 파일 접근 감시
-w /etc/shadow -p wa -k shadow_access

# 네트워크 소켓 생성 감시
-a always,exit -F arch=b64 -S socket -k network_socket
```

### Splunk 연동

Splunk Add-on for Unix and Linux를 사용하면 `auditd` 로그를 쉽게 파싱할 수 있습니다. 로그는 보통 `/var/log/audit/audit.log`에 저장됩니다.

---

## 4. Cloud-Native & eBPF: 새로운 패러다임

클라우드 네이티브 환경, 특히 쿠버네티스(K8s)에서는 기존의 방식이 잘 통하지 않습니다. 여기서 등장한 구세주가 바로 **eBPF(Extended Berkeley Packet Filter)**입니다.

### eBPF란 무엇인가?

eBPF는 리눅스 커널을 수정하거나 모듈을 로드하지 않고도 커널 내에서 샌드박스화된 프로그램을 실행할 수 있게 해주는 기술입니다. 커널 수준에서 발생하는 모든 일을 아주 적은 오버헤드로 관찰할 수 있습니다. 보안 측면에서는 "커널의 눈" 역할을 합니다.

### Falco: eBPF 기반의 런타임 보안

Falco는 eBPF를 사용해 컨테이너와 호스트의 활동을 감시하는 오픈소스 도구입니다.

#### 컨테이너 탈출(Container Escape) 탐지 규칙

공격자가 컨테이너를 뚫고 호스트 OS로 나오려는 시도를 탐지하는 Falco 규칙 예시입니다.

```yaml
- rule: Container Escape via Sensitive Mount
  desc: Detects an attempt to access sensitive host files from a container
  condition: >
    container.id != host and
    fd.name startswith /etc/shadow and
    proc.name != "authorized_proc"
  output: "Sensitive file access from container (user=%user.name command=%proc.cmdline file=%fd.name container_id=%container.id)"
  priority: CRITICAL
```

### Splunk HEC로 알람 전송

Falco의 `falco.yaml` 설정에서 HTTP Output을 활성화해 Splunk HEC(HTTP Event Collector)로 직접 보낼 수 있습니다.

```yaml
http_output:
  enabled: true
  url: "http://splunk-server:8088/services/collector/event"
  user_agent: "falcosecurity/falco"
  insecure: true
  ca_bundle: ""
  ca_path: ""
  ca_file: ""
  mtls: false
  client_cert: ""
  client_key: ""
  headers:
    Authorization: "Splunk <YOUR_HEC_TOKEN>"
```

---

## 5. Splunk SPL을 활용한 상관관계 분석

데이터가 모였다면 이제 분석할 차례입니다.

### Sysmon 프로세스 트리 시각화

특정 프로세스가 어떤 자식 프로세스를 생성했는지 추적하는 SPL입니다.

```splunk
index=endpoint_logs sourcetype="xmlwineventlog" EventCode=1
| table _time, host, ParentImage, Image, CommandLine
| sort _time
| transaction host ParentProcessId ProcessId connected=false
```

### 비정상적인 쉘 실행 탐지

웹 서버 프로세스(apache, nginx)가 갑자기 쉘을 실행하는 경우를 찾는 쿼리입니다.

```splunk
index=endpoint_logs (Image="*sh" OR Image="*cmd.exe")
| where match(ParentImage, "apache|nginx|httpd")
| stats count by host, ParentImage, Image, CommandLine
```

---

## 6. 실습 가이드 (Lab Walkthrough)

1.  **환경 구성**: 윈도우 서버 1대, 우분투 서버 1대, Splunk Enterprise 설치.
2.  **Sysmon 설치**: 윈도우에 Sysmon을 설치하고 SwiftOnSecurity XML을 적용합니다.
3.  **auditd 설정**: 우분투에 `auditd`를 설치하고 위에서 언급한 규칙을 추가합니다.
4.  **Falco 설치**: 우분투에 Docker를 설치하고 Falco를 컨테이너로 실행합니다. eBPF 드라이버를 사용하도록 설정하세요.
5.  **Splunk UF 설치**: 각 서버에 UF를 설치하고 로그를 Splunk로 보냅니다.
6.  **공격 시뮬레이션**: 윈도우에서 파워쉘 인코딩 명령을 실행하고, 리눅스에서 `/etc/shadow` 파일을 읽어보세요.
7.  **대시보드 구축**: Splunk에서 탐지된 이벤트를 시각화하는 대시보드를 만듭니다.

## 마치며

보안은 끝없는 숨바꼭질입니다. 하지만 Sysmon, auditd, Falco와 같은 강력한 도구와 eBPF라는 혁신적인 기술을 활용한다면, 우리는 공격자보다 한발 앞서 나갈 수 있습니다. 오늘 배운 내용을 바탕으로 여러분의 환경에 맞는 최적의 텔레메트리 체계를 구축해 보시기 바랍니다.

궁금한 점은 언제든 댓글로 남겨주세요. 30년 짬바를 녹여 답변해 드리겠습니다.
