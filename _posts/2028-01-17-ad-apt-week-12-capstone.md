---
layout: post
title: "Offensive AD & APT Mastery: Week 12 - Capstone: Full Chain Attack & Defense Simulation"
---

축하합니다. 12주의 대장정이 끝났습니다.
LDAP 조회부터 골든 티켓까지, 여러분은 AD를 공격하고 방어하는 모든 기술을 익혔습니다.

마지막 과제는 여러분이 배운 모든 것을 쏟아부어 **시나리오 기반의 모의해킹**을 수행하는 것입니다.

---

## 🏆 Capstone Project: Operation "Take The Crown"

**목표:** 외부 접점(Initial Access)부터 시작하여 도메인 컨트롤러를 장악(Domain Dominance)하고, 방어자(Blue Team) 관점에서 이를 분석하는 보고서를 작성하세요.

### Scenario
1.  **Target:** 가상의 기업 "Global Corp" (여러분이 구축한 AD 랩).
2.  **Starting Point:** 피싱으로 획득한 `Guest` 권한의 쉘 하나.

### Step 1: Red Teaming (Attack)
다음 단계를 모두 수행하고 스크린샷을 남기세요.
1.  **Recon:** BloodHound로 최단 경로 파악.
2.  **PrivEsc:** Kerberoasting이나 로컬 취약점으로 권한 상승.
3.  **Lateral Movement:** 관리자 PC로 이동하여 해시 탈취.
4.  **Domain Admin:** DC 장악 및 `krbtgt` 해시 덤프.
5.  **Persistence:** Golden Ticket 생성 및 검증.

### Step 2: Blue Teaming (Defense)
공격 수행 후 로그를 분석하세요.
1.  **Timeline:** 공격이 언제 시작되어 언제 끝났는가?
2.  **Artifacts:** 해커가 남긴 파일, 레지스트리 변경, 로그 흔적은?
3.  **Detection:** 만약 실시간 관제 중이었다면 어느 단계에서 잡을 수 있었을까?

### Deliverables (제출물)
*   **Attack Report:** 경영진이 이해할 수 있는 요약 + 엔지니어가 재현할 수 있는 상세 기술 문서.
*   **Defense Recommendations:** GPO 설정 변경, 모니터링 룰 추가 등 구체적인 대응 방안.

---

## 🎓 Closing Remarks

보안 전문가는 '해킹할 줄 아는 사람'이 아니라 **'해킹을 이해하고 막을 줄 아는 사람'**입니다.
공격 기술을 배운 이유는 방어를 더 잘하기 위함임을 잊지 마십시오.

여러분의 기술이 더 안전한 디지털 세상을 만드는 데 쓰이길 바랍니다.
수고하셨습니다.

**Secure the Domain.**
