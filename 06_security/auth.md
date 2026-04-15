# 인증/인가 심화: JWT & OAuth 2.0

## 세션 vs 토큰 기반 인증

### 세션 기반 (Stateful)

```
클라이언트                    서버                      저장소
    │── POST /login ─────────→│                          │
    │                          │── session 생성 ─────────→│
    │                          │   {userId: 1, role: admin}
    │←── Set-Cookie: sid=abc ──│                          │
    │                          │                          │
    │── GET /api (Cookie: abc)→│                          │
    │                          │── 세션 조회 ────────────→│
    │                          │←── {userId: 1, role: admin}
    │←── 응답 ─────────────────│
```

- 서버가 세션 상태를 저장 → **Stateful**
- 서버 스케일아웃 시 세션 공유 필요 (Redis Cluster)
- 즉시 무효화 가능 (서버에서 세션 삭제)
- CSRF 공격에 취약 (쿠키 자동 전송)

### 토큰 기반 (Stateless)

```
클라이언트                    서버
    │── POST /login ─────────→│
    │←── JWT Token ────────────│  (서버는 아무것도 저장 안 함)
    │                          │
    │── GET /api               │
    │   Authorization: Bearer <JWT>
    │                          │
    │                         JWT 서명 검증 (저장소 조회 없음)
    │←── 응답 ─────────────────│
```

- 서버가 상태 저장 안 함 → **Stateless** → 스케일아웃 쉬움
- 토큰 만료 전 즉시 무효화가 어려움 (→ 짧은 만료 + Refresh Token)
- CORS 환경에서 유리 (쿠키 불필요)

---

## JWT (JSON Web Token) 구조 심화

### 토큰 형식

```
Header.Payload.Signature
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjMifQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header** (base64url 디코딩):
```json
{
  "alg": "RS256",    // 서명 알고리즘
  "typ": "JWT",
  "kid": "2024-key-1"   // Key ID (키 로테이션 시 어떤 키로 서명했는지 식별)
}
```

**Payload** (base64url 디코딩):
```json
{
  "iss": "https://auth.example.com",  // Issuer (발급자)
  "sub": "user-123",                   // Subject (사용자 ID)
  "aud": "api.example.com",           // Audience (수신자)
  "exp": 1713200000,                  // Expiration (만료 시각, Unix timestamp)
  "iat": 1713196400,                  // Issued At (발급 시각)
  "jti": "unique-token-id",           // JWT ID (중복 방지)
  "role": "admin",                    // Custom claim
  "scope": "read:users write:posts"   // 권한 범위
}
```

**Signature**:
```
RSASSA-PKCS1-v1_5(
  SHA256(base64url(header) + "." + base64url(payload)),
  privateKey
)
```

### 서명 알고리즘 비교

| 알고리즘 | 종류 | 키 | 검증 가능 주체 |
|---------|------|-----|------------|
| **HS256** | HMAC-SHA256 | 대칭키 (공유 비밀) | 키를 아는 모든 서버 |
| **RS256** | RSA-SHA256 | 비대칭키 (공개키/개인키) | 공개키를 가진 누구나 |
| **ES256** | ECDSA-P256 | 비대칭키 | 공개키를 가진 누구나 |

**마이크로서비스에서 RS256/ES256 권장**:
```
Auth Server: 개인키로 서명 (비밀)
API Server 1: 공개키로 검증만 (Auth Server 호출 불필요)
API Server 2: 공개키로 검증만
API Server N: 공개키로 검증만
```
→ HS256이면 모든 서버가 대칭키를 공유해야 해서 키 노출 위험 증가.

### JWT 취약점

**1. alg: none 공격**
```json
// 변조된 헤더
{"alg": "none", "typ": "JWT"}
// 서명 없이 페이로드 변조 가능
```
→ 방어: 서버에서 `alg` 화이트리스트 검증 필수. `none` 절대 허용 금지.

**2. RS256 → HS256 혼동 공격**
```
서버가 공개키 P로 RS256 검증 설정
공격자: alg를 HS256으로 변경, 공개키 P로 HMAC 서명
서버가 alg를 신뢰하면 → 공개키를 HMAC 키로 사용해 검증 통과
```
→ 방어: 서버에서 허용 알고리즘 명시적 지정.

**3. kid (Key ID) 인젝션**
```json
{"alg": "HS256", "kid": "../../dev/null"}
// 서버가 kid를 파일 경로로 사용하는 경우 빈 파일로 서명 → 빈 문자열로 검증
```

**4. 민감 정보 Payload에 저장 금지**
```
base64url은 인코딩이지 암호화가 아님 → 디코딩 누구나 가능
비밀번호, 개인정보 절대 포함 금지
```

### Access Token + Refresh Token 패턴

```
Access Token:
  - 짧은 만료 (15분~1시간)
  - API 요청마다 사용
  - 무효화 어려움 → 짧은 만료로 보완

Refresh Token:
  - 긴 만료 (7일~30일)
  - Access Token 재발급에만 사용
  - 서버에 저장 (DB/Redis) → 즉시 무효화 가능 (로그아웃, 기기 관리)
  - Refresh Token Rotation: 사용 시 새 RT 발급, 이전 RT 무효화

흐름:
Client → Access Token 만료 →
Client → POST /auth/refresh (Refresh Token) →
Server → Refresh Token 검증 (DB 조회) →
Server → 새 Access Token + 새 Refresh Token 발급
```

---

## OAuth 2.0

### 핵심 역할

```
Resource Owner: 사용자 (리소스 주인)
Client: 서드파티 앱 (카카오 로그인 버튼을 넣은 앱)
Authorization Server: 인증 서버 (카카오 서버)
Resource Server: API 서버 (카카오 API)
```

### Grant Type 1: Authorization Code (가장 일반적, 웹 앱)

```
사용자              클라이언트 앱          Authorization Server
  │                     │                        │
  │ 버튼 클릭           │                        │
  │──────────────────→  │                        │
  │                     │── Redirect ──────────→ │
  │                     │  ?response_type=code    │
  │                     │   &client_id=xxx        │
  │                     │   &redirect_uri=...     │
  │                     │   &state=random123      │  ← CSRF 방지
  │                     │   &code_challenge=xxx   │  ← PKCE (아래 설명)
  │                     │                        │
  │←── 로그인 화면 ─────────────────────────────│
  │── 사용자 동의 ──────────────────────────────→│
  │                     │                        │
  │                     │←── redirect_uri?code=ABC
  │                     │         &state=random123
  │                     │                        │
  │                     │── POST /token ─────────→│
  │                     │  code=ABC               │
  │                     │  client_secret=...      │
  │                     │                        │
  │                     │←── access_token=...    │
  │                     │    refresh_token=...    │
  │                     │    expires_in=3600      │
```

**state 파라미터**: 랜덤 값을 세션에 저장 → 콜백 시 검증 → CSRF 방지

**PKCE (Proof Key for Code Exchange)**:
```
1. 클라이언트가 code_verifier(랜덤 문자열) 생성
2. code_challenge = BASE64URL(SHA256(code_verifier)) 전송
3. 토큰 요청 시 code_verifier 전송
4. 서버가 hash 재계산해서 검증
→ Authorization Code 탈취되어도 code_verifier 없으면 무용지물
→ SPA/모바일에서 client_secret 없이 안전하게 사용 가능
```

### Grant Type 2: Client Credentials (서버 간 통신)

```
클라이언트 서버 → Authorization Server
  POST /token
  grant_type=client_credentials
  client_id=xxx
  client_secret=yyy
  scope=api:read

← access_token (서비스 자체의 토큰, 특정 사용자 없음)
```
- 사용자 개입 없음 → 백엔드 서비스 간 인증에 사용
- Microservices에서 서비스 A → 서비스 B API 호출 시

### OIDC (OpenID Connect)

OAuth 2.0 위에 **인증(Authentication)** 레이어 추가.
OAuth는 인가(Authorization) 프로토콜이지 인증 프로토콜이 아님.

```
OAuth 2.0 응답: access_token (리소스 접근 권한)
OIDC 추가:     id_token (JWT, 사용자 신원 정보)

id_token payload:
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",   // Google의 고유 사용자 ID
  "email": "user@example.com",
  "name": "홍길동",
  "picture": "https://...",
  "aud": "client-id",
  "exp": 1713200000
}
```

**소셜 로그인 ("카카오로 로그인") = OAuth 2.0 + OIDC 조합**

---

## RBAC vs ABAC

### RBAC (Role-Based Access Control)

```
User → Role → Permission

사용자: alice
역할:   admin, editor
권한:
  admin:  create, read, update, delete (모든 리소스)
  editor: read, update (post, comment)

alice가 DELETE /posts/1 요청:
  alice의 역할(admin) → admin 권한에 DELETE 있음 → 허용
```

구현 예:
```python
ROLE_PERMISSIONS = {
    "admin": {"posts": ["create", "read", "update", "delete"]},
    "editor": {"posts": ["read", "update"]},
    "viewer": {"posts": ["read"]},
}

def has_permission(user_roles, resource, action):
    for role in user_roles:
        if action in ROLE_PERMISSIONS.get(role, {}).get(resource, []):
            return True
    return False
```

### ABAC (Attribute-Based Access Control)

```
Subject 속성 + Resource 속성 + Environment 속성 → 정책 평가

예: "부서=회계 AND 데이터_분류=내부 AND 시간=업무시간 → 허용"

AWS IAM 조건 (ABAC의 일종):
{
  "Condition": {
    "StringEquals": {"aws:RequestedRegion": "ap-northeast-2"},
    "IpAddress": {"aws:SourceIp": "203.0.113.0/24"},
    "DateLessThan": {"aws:CurrentTime": "2026-12-31T00:00:00Z"}
  }
}
```

| | RBAC | ABAC |
|--|------|------|
| 복잡도 | 단순 | 복잡 |
| 유연성 | 낮음 | 매우 높음 |
| 성능 | 빠름 (역할 조회) | 느림 (정책 평가) |
| 적합 | 역할이 명확한 시스템 | 세밀한 접근 제어 |

---

## 실무 포인트

- **Access Token은 메모리에만 저장** (XSS 방어): `localStorage`/`sessionStorage` 금지. `httpOnly` 쿠키에 Refresh Token 보관
- **토큰 블랙리스트**: 로그아웃/기기 변경 시 Refresh Token을 Redis 블랙리스트에 추가 → Access Token 만료 전까지는 유효하지만 갱신 불가
- **JWT 크기 주의**: 모든 요청에 포함 → 너무 많은 claim은 대역폭 낭비. 역할/권한은 간략하게, 상세 정보는 필요 시 API 조회
- **JWKS (JSON Web Key Set)**: `/oauth/.well-known/jwks.json`으로 공개키 제공. 키 로테이션 시 `kid`로 구분하여 점진 교체 가능
- **토큰 만료 처리**: `exp` 검증은 시계 오차 허용 (`leeway` 몇 초). 서버 간 시각 동기화 (NTP) 필수
- **OAuth PKCE 필수**: SPA, 모바일 앱에서 Authorization Code Flow 사용 시 PKCE 없으면 Code Interception Attack 취약
- **SSO (Single Sign-On)**: OIDC 기반. 한 번 인증으로 여러 서비스 접근. Keycloak, Okta, AWS Cognito가 구현체
