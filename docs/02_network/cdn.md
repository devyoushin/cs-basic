# CDN (Content Delivery Network)

## CDN이란

지리적으로 분산된 엣지 서버 네트워크를 통해 콘텐츠를 사용자에게 더 빠르고 안정적으로 전달하는 인프라.

```
[Origin Server] ←── 원본 콘텐츠 보관
       ↑
   (캐시 미스 시만 요청)
       ↑
[Edge Server (PoP)] ←── 사용자와 물리적으로 가까운 서버
       ↑
[사용자]  ←── RTT 대폭 감소, Origin 부하 감소
```

**PoP (Point of Presence)**: 엣지 서버가 위치한 데이터센터. Cloudflare 300+, Akamai 4000+

---

## CDN 동작 원리 — 요청 흐름

### 1. DNS 기반 트래픽 유입

```
사용자: www.example.com 요청
  └→ DNS 조회: www.example.com
        └→ CNAME: example.cdn-provider.net
              └→ Anycast/GeoDNS → 가장 가까운 PoP IP 반환
사용자 → 해당 PoP로 HTTP 요청
```

CDN은 DNS 계층에서부터 트래픽을 낚아챔. CNAME으로 위임하거나, Auth NS 자체를 CDN이 운영.

### 2. 엣지 캐시 처리 흐름

```
사용자 요청 →  엣지 서버
                │
                ├─ Cache HIT  → 즉시 응답 (Origin 요청 없음)
                │
                └─ Cache MISS → Origin 요청
                                 → 응답 수신
                                 → 캐시 저장 (Cache-Control 헤더 기반)
                                 → 사용자에게 응답
```

### 3. Anycast vs GeoDNS 라우팅

| 방식 | 원리 | 특징 |
|------|------|------|
| **Anycast** | 동일 IP를 BGP로 여러 PoP에서 광고 → 라우터가 최단 경로 선택 | IP 변경 없이 트래픽 분산. DDoS 흡수에 강함 |
| **GeoDNS** | 질의자 IP 기반으로 가까운 PoP IP 반환 | DNS TTL로 인한 반응 지연. ECS로 정확도 향상 |

Cloudflare는 Anycast, AWS CloudFront는 GeoDNS + Anycast 혼용.

---

## CDN 캐싱 전략

### Pull CDN (가장 일반적)
- Origin에서 콘텐츠를 가져와 캐싱 (캐시 미스 시 자동 Pull)
- 처음 요청한 사용자만 느리고, 이후는 캐시 히트
- CloudFront, Cloudflare, Fastly가 기본적으로 Pull 방식

### Push CDN
- 운영자가 사전에 엣지 서버로 콘텐츠를 업로드
- 대용량 파일(게임 패치, 미디어 파일) 배포에 적합
- 캐시 미스 자체가 없음. 단, 스토리지 비용 발생

### Cache Key 구성
```
기본: 프로토콜 + 호스트 + 경로
예: https://example.com/api/v1/products

확장: 쿼리스트링, 헤더, 쿠키 포함 가능
예: https://example.com/api/v1/products?lang=ko  (lang 쿼리 포함 시 별도 캐시)
```

**주의**: 쿼리스트링을 Cache Key에 포함하면 캐시 분산 → HIT율 저하. 불필요한 파라미터 제거 필요.

### Cache-Control 헤더 동작

```http
Cache-Control: max-age=3600                  # 1시간 캐시 (브라우저 + CDN)
Cache-Control: s-maxage=86400, max-age=0     # CDN 24시간, 브라우저 캐시 없음
Cache-Control: no-store                      # CDN/브라우저 모두 캐싱 금지
Cache-Control: private                       # 브라우저만 캐시, CDN 캐싱 금지
Cache-Control: stale-while-revalidate=60     # 만료 후 60초간 이전 캐시 서빙하며 백그라운드 갱신
Cache-Control: stale-if-error=86400         # Origin 에러 시 24시간 이전 캐시 사용
```

### CDN 캐시 무효화 (Invalidation)
- **CloudFront**: Invalidation API (`/*` 는 비용 발생)
- **Cloudflare**: Cache Purge API (무료, 즉시 적용)
- **Versioned URL**: `/static/app.v3.js` — 파일명에 버전 포함 → 무효화 없이 새 캐시

```bash
# Cloudflare Cache Purge 예시
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
     -H "Authorization: Bearer {token}" \
     -d '{"files":["https://example.com/api/products"]}'
```

---

## CDN 캐시 계층 구조

```
[사용자]
   │
[엣지 서버 (L1 캐시)] ← 수백~수천 개, 사용자와 가장 가까움
   │ 미스
[오리진 쉴드 (L2 캐시)] ← PoP 중간 집결지, Origin 보호
   │ 미스
[Origin Server] ← CDN 없으면 모든 요청이 여기로
```

### Origin Shield (미드티어 캐시)
- 여러 PoP의 캐시 미스를 한 지점에서 집결 → Origin 요청을 대폭 감소
- CloudFront: Origin Shield 기능 제공
- 글로벌 PoP가 많을수록 Origin 요청 중복 방지 효과 큼

---

## CDN과 TLS/HTTPS

### TLS 종료 위치

```
[사용자] ──TLS──> [엣지 서버] ──HTTP or TLS──> [Origin]
```

- 엣지에서 TLS 종료 → 사용자와 가장 가까운 곳에서 TLS 핸드셰이크 → RTT 감소
- Origin까지 재암호화: 보안 강화 (End-to-End TLS)
- Origin까지 HTTP: 내부 네트워크 신뢰 (성능 향상, 보안 위험)

### TLS 성능 최적화
- **Session Resumption**: TLS Session Ticket / Session ID로 핸드셰이크 단축
- **OCSP Stapling**: 엣지가 인증서 유효성 응답을 캐싱 → 클라이언트의 CA 쿼리 제거
- **TLS 1.3**: 1-RTT 핸드셰이크 (0-RTT Early Data 지원 → Replay Attack 주의)

---

## 동적 콘텐츠와 CDN

### 캐싱 불가한 동적 콘텐츠 처리

| 방법 | 설명 |
|------|------|
| **CDN 바이패스** | 동적 API는 CDN 우회, 정적 자산만 CDN 경유 |
| **Edge Computing** | CDN 엣지에서 코드 실행 → 동적 처리를 Origin 없이 |
| **Partial Caching** | 페이지 일부만 캐시 (ESI: Edge Side Includes) |
| **SWR 패턴** | stale-while-revalidate로 캐시 히트 유지하며 백그라운드 갱신 |

### Edge Computing (CDN의 확장)
```
[사용자] → [엣지 서버 (코드 실행)] → 필요 시 Origin
```

- **Cloudflare Workers**: V8 엔진 기반, JS/WASM 실행 (콜드 스타트 ~0ms)
- **AWS Lambda@Edge / CloudFront Functions**: 요청/응답 조작
- **Fastly Compute@Edge**: WASM 기반

**활용 사례**
- A/B 테스트 트래픽 분기
- 인증 토큰 검증 (Origin 부하 제거)
- 지역 기반 리다이렉트
- Request/Response 헤더 조작

---

## CDN과 보안

### DDoS 방어
- Anycast로 공격 트래픽을 전 세계 PoP에 분산 흡수
- Rate Limiting: IP/UA 기반 요청 수 제한
- IP Reputation: 악성 IP 자동 차단
- Cloudflare Magic Transit: L3/L4 레벨 DDoS 방어 (BGP 기반)

### WAF (Web Application Firewall)
- CDN 엣지에서 L7 공격 차단 (SQLi, XSS, CSRF 등)
- OWASP Top 10 룰셋 기반 필터링
- Origin 도달 전에 차단 → Origin 보호

### Origin IP 숨기기
```
사용자 → CDN → Origin (프라이빗 IP 또는 CDN만 알고 있는 IP)
```
- Origin IP 노출 시 CDN 우회 공격 가능
- Cloudflare Tunnel / AWS PrivateLink로 Origin을 완전히 숨길 수 있음
- `CF-Connecting-IP`, `X-Forwarded-For` 헤더로 실제 클라이언트 IP 확인

---

## CDN 성능 지표

| 지표 | 의미 | 목표 |
|------|------|------|
| **Cache HIT Rate** | 전체 요청 중 캐시 히트 비율 | 90% 이상 |
| **TTFB (Time to First Byte)** | 첫 바이트 수신까지 시간 | < 200ms |
| **Cache HIT TTFB** | 캐시 히트 시 응답 속도 | < 50ms |
| **Origin Offload** | Origin 요청 절감 비율 | 높을수록 좋음 |

---

## CDN 진단

```bash
# 캐시 상태 확인 (응답 헤더)
curl -I https://example.com/image.jpg
# X-Cache: HIT         → 캐시 히트
# X-Cache: MISS        → 캐시 미스, Origin에서 가져옴
# CF-Cache-Status: HIT → Cloudflare 캐시 히트
# Age: 3600            → 캐시된 지 3600초 경과

# 여러 PoP에서 동일 URL 테스트
# https://www.whatsmydns.net/ — DNS 전파 확인
# https://check-host.net/     — 글로벌 HTTP 응답 확인
```

---

## 실무 포인트

- **Cache HIT율 먼저 확인**: CDN 도입 후 HIT율이 낮으면 Cache Key 설정 검토 (불필요한 쿼리스트링 제거)
- **Origin Shield 활용**: 글로벌 서비스라면 Origin 앞에 Shield 레이어 추가 → Origin 요청 수 10배 이상 감소 가능
- **Versioned URL 필수**: `app.js?v=123` 보다 `app.v123.js` — 쿼리스트링 캐시는 CDN마다 동작이 달라 위험
- **동적 API는 캐시 우회**: `Cache-Control: no-store` or `private` 명시. 또는 도메인 분리 (`api.example.com`은 CDN 제외)
- **WAF 규칙 튜닝**: 지나치게 엄격한 WAF 룰은 정상 트래픽 차단 → 처음엔 모니터링 모드로 운영
- **CDN Failover**: Origin 장애 시 `stale-if-error` 로 이전 캐시 서빙 → 사용자 경험 보호
- **실제 클라이언트 IP**: 로그/방화벽에서 `X-Forwarded-For` or `CF-Connecting-IP` 헤더 사용. CDN IP 범위를 신뢰 목록에 추가
- **비용 주의**: CDN은 전송량(egress) 기준 과금. 대용량 파일 캐시 HIT율이 낮으면 오히려 Origin 직접 전송보다 비쌀 수 있음
