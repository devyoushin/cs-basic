# HTTP / HTTPS

## HTTP/1.1 메시지 구조

### 요청(Request) 메시지
```http
POST /api/users HTTP/1.1                ← 요청 라인: 메서드 + URI + 버전
Host: api.example.com                   ← 헤더
Content-Type: application/json
Content-Length: 27
Connection: keep-alive

{"name": "Alice", "age": 30}            ← 바디 (빈 줄로 헤더와 구분)
```

### 응답(Response) 메시지
```http
HTTP/1.1 200 OK                         ← 상태 라인: 버전 + 상태코드 + 이유 문구
Content-Type: application/json
Content-Length: 42
Date: Mon, 15 Apr 2026 10:00:00 GMT

{"id": 1, "name": "Alice", "age": 30}   ← 바디
```

### HTTP/1.1 Keep-Alive
```http
Connection: keep-alive     ← TCP 연결 재사용 (기본값)
Connection: close          ← 응답 후 TCP 연결 종료
Keep-Alive: timeout=5, max=100
```
- HTTP/1.0: 요청마다 새 TCP 연결 (느림)
- HTTP/1.1: Keep-Alive로 TCP 재사용. 그러나 **파이프라이닝은 HOL Blocking 문제**

### HTTP/1.1 파이프라이닝 문제 (HOL Blocking)
```
요청: GET /a → GET /b → GET /c  (순서대로 전송)
응답: /a (느림) ... /b ... /c
         ↑ /a가 느리면 /b, /c 응답도 기다려야 함
```

---

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

### HTTP/2 주요 기능

**멀티플렉싱 (Multiplexing)**
```
HTTP/1.1: 요청 → 응답 → 요청 → 응답 (순차)
HTTP/2:   요청1 ──────────→ 응답1
          요청2 ────→ 응답2        (동시 병렬)
          요청3 ──→ 응답3
          ↑ 하나의 TCP 연결에서 여러 스트림 병렬 처리
```

**프레임(Frame) 구조**
- HTTP/2는 텍스트 → 바이너리 프레임으로 변환
- 프레임 타입: DATA, HEADERS, SETTINGS, PING, RST_STREAM, GOAWAY 등
- 각 스트림은 고유 Stream ID를 가짐 (클라이언트: 홀수, 서버: 짝수)

**HPACK 헤더 압축**
- 정적 테이블 (61개 미리 정의된 헤더) + 동적 테이블 사용
- 반복 헤더를 인덱스로 대체 → 60~90% 크기 감소

**서버 푸시 (Server Push)**
```
클라이언트: GET /index.html
서버: index.html + (푸시) style.css + (푸시) main.js
→ 클라이언트 요청 전에 리소스 미리 전송 (현재 Chrome에서 비활성화 추세)
```

### HTTP/3 & QUIC

```
HTTP/1.1 & HTTP/2: TCP 기반
HTTP/3: QUIC (UDP 기반) 사용
```

**QUIC 핵심 특징**

| 항목 | TCP+TLS | QUIC |
|------|---------|------|
| 연결 수립 | 3-way handshake + TLS handshake (2-3 RTT) | 1 RTT (0-RTT 재연결) |
| HOL Blocking | 발생 (패킷 손실 시 전체 블로킹) | 없음 (스트림 독립) |
| 연결 마이그레이션 | IP 변경 시 연결 끊김 | Connection ID로 유지 |
| 암호화 | 선택적 (TLS) | 필수 내장 |

**0-RTT 재연결**
```
최초 연결: 1 RTT (QUIC Handshake)
재연결:    0 RTT (이전 세션 정보 재사용 → 즉시 데이터 전송)
```
단점: Replay Attack 위험 (幂等하지 않은 요청에는 사용 불가)

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
- 취약한 암호화 알고리즘 제거 (RC4, 3DES, MD5 등)
- Forward Secrecy 필수 (세션 키 노출되어도 과거 트래픽 복호화 불가)

### SNI (Server Name Indication)
하나의 IP에 여러 도메인의 TLS 인증서를 제공:
```
클라이언트: ClientHello + SNI: "api.example.com"
서버: SNI를 보고 적절한 인증서 선택 후 제공
```
- 가상 호스팅(Virtual Host)에서 HTTPS를 가능하게 하는 TLS 확장
- Nginx: `server_name api.example.com;` 기반으로 인증서 자동 선택
- 문제: SNI는 평문으로 전송 → 도청 가능. **ECH (Encrypted Client Hello)**로 해결 예정

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
