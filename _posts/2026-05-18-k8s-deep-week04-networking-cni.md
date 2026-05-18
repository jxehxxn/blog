---
layout: post
title: "K8s Deep Week 4: Networking — CNI, IPAM, Pod-to-Pod 패킷 추적"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series networking
---

## 학습 목표

- K8s 네트워크 3원칙을 비유로 안다.
- CNI 플러그인이 Pod에 IP를 부여하는 과정을 안다.
- Pod-to-Pod 패킷이 노드를 건너가는 두 가지 방식(L2 vs Overlay)을 이해한다.
- 직접 패킷을 tcpdump로 추적해 본다.

## 1. K8s 네트워크 3원칙

학부생을 위한 한 문장: **"같은 캠퍼스 안의 모든 학생(Pod)은 학번(IP)만 알면 서로 직접 편지를 보낼 수 있어야 한다. 우체국(NAT) 거치지 않고."**

기술적 원칙:
1. 모든 Pod은 클러스터 전체에서 unique한 IP를 가진다.
2. 모든 Pod은 NAT 없이 서로 직접 통신 가능하다.
3. 노드도 NAT 없이 Pod과 통신 가능하다.

이게 충족되도록 네트워크를 구성하는 것이 **CNI 플러그인**의 책임입니다.

## 2. CNI — Container Network Interface

비유: 새 학생(Pod)이 캠퍼스에 입학할 때마다, 행정실(CNI)이 (1) 학번 발급(IP 할당), (2) 강의실 문에 이름표 부착(veth pair), (3) 캠퍼스 전화번호부 갱신(route table) 합니다.

CNI는 "표준"입니다. 구현체는 Calico, Cilium, Flannel, Weave 등 여러 가지.

CNI 흐름:

```
[kubelet]
   ↓ "Pod 만들었으니 네트워크 설정해줘"
[CNI plugin binary] (/opt/cni/bin/calico 등)
   ↓ stdin으로 JSON 전달 (Pod ID, namespace 등)
   1. veth pair 만듦 (한 쪽 host, 한 쪽 container)
   2. container 쪽 veth에 IP 할당 (IPAM)
   3. host 쪽 veth를 bridge 또는 routing table에 연결
   ↓ stdout으로 결과 JSON 반환
[kubelet] ← Pod이 네트워크 사용 가능
```

`/etc/cni/net.d/` 안에 어떤 CNI를 쓸지 conf 파일.

## 3. IPAM — IP Address Management

캠퍼스에 학번을 발급하는 방식 결정:
- **Host-local**: 각 노드가 "내 노드에서 쓸 IP 풀"을 미리 받고, 자기가 알아서 할당. 단순.
- **DHCP-like**: 중앙 서버가 IP를 부여. K8s에선 흔치 않음.
- **외부 IPAM**: Cilium의 Cluster Pool, Calico의 IPPool 등.

`kubectl get nodes -o yaml | grep podCIDR` 로 각 노드의 IP 풀 확인.

```yaml
spec:
  podCIDR: 10.244.0.0/24       # worker-3 노드는 10.244.0.x 사용
```

## 4. Pod-to-Pod 패킷 추적 — 두 가지 시나리오

### 시나리오 A: 같은 노드의 두 Pod

```
[Pod A: 10.244.0.5]
       ↓ ping 10.244.0.6
   [eth0 (veth A)]
       ↓
   [host 쪽 veth A]
       ↓
   [노드의 bridge cni0]
       ↓
   [host 쪽 veth B]
       ↓
   [eth0 (veth B)]
[Pod B: 10.244.0.6]
```

같은 노드의 Linux bridge(`cni0`)가 L2 스위치 역할. NAT 없음.

### 시나리오 B: 다른 노드의 두 Pod

여기서 갈립니다.

#### (B1) L2 direct routing (Calico의 BGP 모드)

```
[Pod A on worker-3: 10.244.0.5]
       ↓
   [worker-3의 routing table]
       "10.244.1.0/24 via 192.168.1.4 (= worker-4의 노드 IP)"
       ↓
   물리 네트워크 (또는 L3 라우터)
       ↓
   [worker-4]
       ↓ routing
[Pod B on worker-4: 10.244.1.6]
```

각 노드가 자신의 podCIDR을 BGP로 광고. 라우터가 알아서 라우팅. 가장 빠름, 단 물리 네트워크가 BGP를 지원해야.

#### (B2) Overlay (Flannel의 VXLAN)

```
[Pod A on worker-3]
       ↓
   [worker-3 VXLAN encap]
       원본 패킷을 UDP 안에 포장
       ↓
   물리 UDP 패킷 (외부 라우터엔 그냥 UDP로 보임)
       ↓
   [worker-4 VXLAN decap]
       UDP 벗기고 원본 패킷 꺼냄
       ↓
[Pod B on worker-4]
```

물리 네트워크 변경 없이 동작. 단 encap/decap 오버헤드 (5~10% 성능 손실).

CNI 3종 비교는 [보충 2: CNI 비교] 포스트.

## 5. NetworkPolicy — "방화벽"

비유: 캠퍼스에 "공대 학생은 의대 강의실에 못 들어간다" 같은 규칙. NetworkPolicy로 Pod 간 트래픽을 라벨 기반으로 제어.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
spec:
  podSelector: {}        # 이 namespace의 모든 Pod
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}  # 같은 namespace의 Pod만 허용
```

NetworkPolicy는 **CNI가 enforcement** 합니다. Flannel은 NetworkPolicy 지원 안 함 → 보안 강한 환경은 Calico/Cilium 권장.

## 6. eBPF 기반 차세대 — Cilium

비유: 기존 iptables는 "강의실 문마다 검문" 식이라 문이 많아지면 느려집니다. **eBPF**는 "강의실 입구 천장에 X-ray를 달아 모든 출입을 한꺼번에 검사" — 커널 레벨에서 효율적으로 처리.

Cilium은 eBPF로:
- kube-proxy를 대체 (Service 라우팅을 iptables 대신 eBPF).
- NetworkPolicy를 L7(HTTP path, gRPC method)까지 검사.
- 관측성(Hubble) 강화.

빅테크에서 점점 표준이 되어 가는 중. AWS EKS도 Cilium 옵션 추가.

## 7. 실습 — 직접 패킷 추적

```bash
# 1. nginx Pod 2개를 다른 노드에 띄움
kubectl run nginx-a --image=nginx
kubectl run nginx-b --image=nginx
kubectl get pod -o wide
# IP 와 노드 확인

# 2. nginx-a 에서 nginx-b로 curl
kubectl exec nginx-a -- curl 10.244.1.6

# 3. 노드에 들어가 tcpdump
docker exec -it learn-worker bash
tcpdump -i any -n host 10.244.0.5 -c 5

# 4. CNI bridge 확인
ip link show cni0  # bridge
ip route          # podCIDR routes

# 5. iptables 마법 보기
iptables -t nat -L KUBE-SERVICES -n | head
```

## 8. 자가평가 퀴즈

### Q1. K8s 네트워크 3원칙의 핵심?
1. 모든 통신 NAT 사용
2. **모든 Pod이 unique IP, NAT 없이 직접 통신**
3. 모든 Pod이 같은 IP
4. 무관

**정답: 2.**

### Q2. CNI plugin의 책임?
1. Pod이 생성될 때 IP 할당, veth 설정, route 갱신
2. 컨테이너 빌드
3. UI
4. 무관

**정답: 1.**

### Q3. VXLAN overlay의 트레이드오프는?
1. 물리망 변경 불필요 / encap 오버헤드 5~10%
2. 무료
3. 더 빠름
4. 무관

**정답: 1.**

### Q4. NetworkPolicy를 enforce하는 주체는?
1. API server
2. **CNI plugin (Calico/Cilium)**
3. kubelet
4. scheduler

**정답: 2.** Flannel은 enforce 안 함.

### Q5. Cilium이 iptables 대신 eBPF를 쓰는 이유?
1. 무관
2. **수천 Service에서 더 효율적, L7 검사도 가능**
3. UI 색
4. 비용

**정답: 2.**

## 9. 다음 주차

[Week 5: Service / Ingress / Gateway API]에서는 Service의 4종류와 Ingress, 그리고 차세대 Gateway API를 다룹니다.
