---
layout: post
title: "Mesh 최종 자가평가 — 30문항 종합 (정답+해설)"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series assessment
---

12주 학습자 종합 평가.

## Q1. Service mesh 본질?
1. **sidecar proxy로 통신 기능 통합** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q2. Istio data plane?
1. **Envoy** 2. Rust 3. eBPF 4. 무관

**정답: 1.**

## Q3. 5 xDS?
1. **LDS/RDS/CDS/EDS/SDS** 2. UI 3. 4가지 4. 무관

**정답: 1.**

## Q4. Sidecar injection 방법?
1. **namespace label istio-injection=enabled** 2. 수동 3. UI 4. 무관

**정답: 1.**

## Q5. VirtualService 책임?
1. **트래픽 라우팅** 2. LB 3. 인증 4. 무관

**정답: 1.**

## Q6. DestinationRule 책임?
1. **subset, LB, connection pool, outlier** 2. 라우팅 3. UI 4. 무관

**정답: 1.**

## Q7. Sidecar resource 가치?
1. **Envoy 메모리 최적** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q8. Traffic Mirror?
1. **production 트래픽 shadow copy** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q9. Fault Injection?
1. **chaos engineering** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q10. Outlier detection?
1. **연속 실패 endpoint ejection** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q11. Locality LB?
1. **같은 region 우선** 2. UI 3. 무관 4. 빠른 빌드

**정답: 1.**

## Q12. 자동 mTLS 인증서?
1. **SPIFFE 24h 자동 회전** 2. 영구 3. UI 4. 무관

**정답: 1.**

## Q13. STRICT vs PERMISSIVE?
1. **STRICT mTLS만, PERMISSIVE plaintext도** 2. 같음 3. 반대 4. 무관

**정답: 1.**

## Q14. AuthZ zero-trust?
1. **deny by default + allow whitelist** 2. allow only 3. UI 4. 무관

**정답: 1.**

## Q15. JWT 검증 위치?
1. **RequestAuthentication** 2. PeerAuth 3. DestinationRule 4. 무관

**정답: 1.**

## Q16. Istio 자동 metric?
1. **RED 자동** 2. app instrument 필요 3. UI 4. 무관

**정답: 1.**

## Q17. Kiali 가치?
1. **service graph 시각화** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q18. Telemetry CR 가치?
1. **sampling/cardinality 세밀 제어** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q19. Egress Gateway?
1. **외부 IP 통일 + audit** 2. UI 3. 무관 4. 빠름

**정답: 1.**

## Q20. ServiceEntry 용도?
1. **외부 service mesh 등록** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q21. Ambient ztunnel 역할?
1. **per node L4 mTLS** 2. L7 3. UI 4. 무관

**정답: 1.**

## Q22. Waypoint 사용?
1. **L7 정책 필요한 namespace만** 2. 모든 곳 3. 무관 4. UI

**정답: 1.**

## Q23. Primary-Remote vs Multi-Primary?
1. **단순 vs 격리** 2. 같음 3. 무관 4. UI

**정답: 1.**

## Q24. east-west gateway?
1. **cluster 간 mTLS tunnel** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q25. Canary upgrade 핵심?
1. **revision 라벨 점진 이주** 2. 한 번에 3. UI 4. 무관

**정답: 1.**

## Q26. istioctl analyze?
1. **설정 오류 사전 검증** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q27. Linkerd 강점?
1. **가벼움·단순** 2. 기능 풍부 3. UI 4. 무관

**정답: 1.**

## Q28. Istio 강점?
1. **기능 풍부 + 생태계** 2. 가벼움 3. UI 4. 무관

**정답: 1.**

## Q29. Cilium Service Mesh?
1. **eBPF + CNI 통합** 2. sidecar 강조 3. UI 4. 무관

**정답: 1.**

## Q30. 선택 시 핵심 trade-off?
1. **운영 부담 vs 기능** 2. UI 3. 비용 4. 무관

**정답: 1.**

## 채점

- 28~30: Senior 수준.
- 24~27: 미드.
- 18~23: 주니어.
- ~17: 재복습.

다음 코스: OPA/Kyverno.
