---
layout: post
title: "Offensive AD & APT Mastery: Supplement - Hybrid Identity Attacks (Azure AD / Entra ID)"
---

"온프레미스 AD는 털었는데, 클라우드(Azure)는 안전한가요?"
아니요. **AD Connect** 서버를 장악하면 클라우드도 끝입니다.

대부분의 기업은 온프레미스 AD의 계정을 Azure AD로 동기화해서 씁니다. 이 연결 고리를 끊거나 악용하는 것이 **Hybrid Attack**의 핵심입니다.

---

## 1. Azure AD Connect Recon

**AD Connect** 서버는 온프레미스 계정 정보를 클라우드로 복제해주는 서버입니다.
이 서버에는 **"MSOL"**이라는 이름으로 시작하는 서비스 계정이 있는데, 이 계정은 **복제 권한(DCSync)**을 가지고 있습니다.

즉, **AD Connect 서버의 로컬 관리자 권한**만 얻으면?
1.  AD Connect DB에서 MSOL 계정의 비번을 복호화합니다.
2.  그 비번으로 온프레미스 AD 전체를 DCSync 합니다.
3.  Game Over.

---

## 2. Password Hash Sync vs Pass-Through Authentication

*   **PHS (Password Hash Sync):** 온프레미스 해시를 클라우드로 복사. (클라우드만 털리면 온프레미스도 위험)
*   **PTA (Pass-Through):** 클라우드 로그인을 온프레미스 에이전트가 검증. (온프레미스 에이전트에 백도어 심으면 모든 클라우드 로그인 가로채기 가능)

---

## 3. Golden SAML

SolarWinds 해킹 사건 때 쓰였던 전설적인 기법입니다.
온프레미스 AD FS(Federation Services) 서버를 장악했을 때 사용합니다.

1.  **AD FS 서명 키(Token Signing Certificate)**를 훔칩니다.
2.  이 키를 이용해 **"나는 Office 365 관리자다"**라는 SAML 토큰을 위조합니다.
3.  Azure AD는 이 토큰을 믿고 로그인시켜줍니다. (MFA도 우회!)
4.  비밀번호를 몰라도 클라우드 관리자가 됩니다.

---

## 4. Tools: AADInternals & RoadTools

### AADInternals (PowerShell)
Azure AD 해킹의 맥가이버 칼입니다.
```powershell
# AD Connect 서버에서 자격 증명 추출
Get-AADIntSyncCredentials -Extract

# 사용자 열거
Invoke-AADIntUserEnumerationAsGuest
```

---

## 5. Defense Strategy

1.  **Tier Model:** AD Connect 서버는 도메인 컨트롤러(Tier 0)와 동급으로 취급하고 보호해야 합니다.
2.  **Monitor Sync:** Azure AD Connect의 동기화 로그와 서비스 계정의 로그인을 감시해야 합니다.
3.  **Hybrid Join:** 무분별한 하이브리드 조인을 막고, 조건부 액세스(Conditional Access) 정책을 강화해야 합니다.

온프레미스와 클라우드는 이제 한 몸입니다. 하나가 죽으면 다 죽습니다.
