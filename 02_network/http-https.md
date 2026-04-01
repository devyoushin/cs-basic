# HTTP / HTTPS

## HTTP 버전 비교

| 버전 | 특징 | 문제점 |
|------|------|--------|
| HTTP/1.0 | 요청마다 새 TCP 연결 | 매우 느림 |
| HTTP/1.1 | Keep-Alive (연결 재사용), 파이프라이닝 | HOL Blocking |
| HTTP/2 | 멀티플렉싱, 헤더 압축(HPACK), 서버 푸시 | TCP HOL Blocking |
| HTTP/3 | QUIC(UDP 기반), 연결 마이그레이션 | 방화벽 UDP 차단 이슈 |

### HOL (Head-of-Line) Blocking
- HTTP/1.1: 앞의 요청이 완료돼야 다음 응답 처리 가능
- HTTP/2: TCP 레벨에서는 여전히 패킷 손실 시 블로킹 발생
- HTTP/3 (QUIC): 스트림별 독립 처리로 완전 해결

---

## HTTP 메서드

| 메서드 | 용도 | 멱등성 | 안전성 |
|--------|------|--------|--------|
| GET | 조회 | O | O |
| POST | 생성 | X | X |
| PUT | 전체 수정/대체 | O | X |
| PATCH | 부분 수정 | X | X |
| DELETE | 삭제 | O | X |
| HEAD | 헤더만 조회 | O | O |
| OPTIONS | 지원 메서드 확인 | O | O |

- **멱등성**: 여러 번 호출해도 결과 동일
- **안전성**: 서버 상태를 변경하지 않음

---

## HTTP 상태 코드

| 범위 | 의미 | 주요 코드 |
|------|------|---------|
| 1xx | 정보 | 100 Continue |
| 2xx | 성공 | 200 OK, 201 Created, 204 No Content |
| 3xx | 리다이렉션 | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | 클라이언트 오류 | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | 서버 오류 | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### DevOps 관점 주요 코드
- **502 Bad Gateway**: 업스트림 서버(앱서버)가 죽었거나 응답 불가
- **503 Service Unavailable**: 서버 과부하 또는 유지보수 중
- **504 Gateway Timeout**: 업스트림 서버가 시간 내에 응답 안 함
- **429 Too Many Requests**: Rate Limit 초과

---

## HTTP 헤더

### 요청 헤더
```http
Host: api.example.com
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
User-Agent: Mozilla/5.0...
X-Forwarded-For: 203.0.113.1  ← 클라이언트 실제 IP (프록시 통과 시)
X-Request-ID: abc-123         ← 요청 추적용
```

### 응답 헤더
```http
Content-Type: application/json; charset=utf-8
Cache-Control: max-age=3600, public
ETag: "33a64df5"
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Strict
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### 캐시 제어
```
Cache-Control: no-store          # 캐시 완전 금지
Cache-Control: no-cache          # 캐시 저장하되 서버 검증 후 사용
Cache-Control: max-age=3600      # 3600초 캐시
Cache-Control: s-maxage=3600     # CDN/프록시 캐시 시간
ETag + If-None-Match             # 조건부 요청 (변경 없으면 304)
Last-Modified + If-Modified-Since
```

---

## HTTPS & TLS

### TLS Handshake (1.2)
```
Client                      Server
  | ── ClientHello ───────→ |  (지원 암호화 목록)
  | ←─ ServerHello ──────── |  (선택된 암호화)
  | ←─ Certificate ──────── |  (서버 인증서)
  | ── ClientKeyExchange ──→ |  (세션 키 교환)
  | ── Finished ───────────→ |
  | ←─ Finished ─────────── |
  |   [암호화 통신 시작]     |
```

### TLS 1.3 개선점
- Handshake 1 RTT (1.2는 2 RTT)
- 0-RTT 재연결 지원
- 취약한 암호화 알고리즘 제거

### 인증서

```bash
# 인증서 만료일 확인
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# 인증서 상세 정보
openssl x509 -in cert.pem -text -noout

# CSR 생성
openssl req -new -key private.key -out request.csr
```

### 인증서 체인
```
Root CA (자가 서명, 브라우저에 내장)
  └── Intermediate CA
        └── Server Certificate (실제 사이트)
```

---

## CORS (Cross-Origin Resource Sharing)

```
Origin: 프로토콜 + 도메인 + 포트

https://app.example.com → https://api.example.com  ← Cross-Origin
```

### Preflight 요청 (OPTIONS)
브라우저가 실제 요청 전에 서버에 허용 여부 확인:
```http
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

서버 응답:
```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400
```

---

## 실무 포인트

- **X-Forwarded-For vs REMOTE_ADDR**: 로드밸런서/프록시 뒤에서는 XFF 헤더로 클라이언트 IP 파악
- **Keep-Alive 타임아웃**: nginx `keepalive_timeout`이 ALB/NLB 타임아웃보다 길어야 함 (그렇지 않으면 502 발생)
- **TLS 인증서 자동 갱신**: Let's Encrypt + Certbot. `0 0 1 * * certbot renew` 크론
- **HSTS (HTTP Strict Transport Security)**: 브라우저가 HTTPS만 사용하도록 강제. Preload 목록 등재 가능
- **gzip/brotli 압축**: 텍스트 기반 응답 60~80% 크기 감소
- **HTTP/2 사전 조건**: TLS 필수 (대부분의 브라우저), nginx `listen 443 ssl http2`
