# Proxy 완전 분석 — Forward/Reverse/Transparent부터 서비스 메쉬 Sidecar까지

> SRE/DevOps Senior 레벨: 프록시가 패킷 수준에서 어떻게 동작하는지, 실무에서 어떤 문제를 풀어주는지 완전 분석

---

## 1. 프록시란 무엇인가

프록시(Proxy)는 **클라이언트와 서버 사이에서 트래픽을 대리(代理) 처리하는 중개자**다.
단순 전달이 아니라, 연결을 **종료(terminate)** 하고 새 연결을 **재개(originate)** 한다는 점이 핵심이다.

```
[NAT 라우터] — 패킷을 수정해 전달  (IP/Port 변환만, L3/L4)
[프록시]     — 연결을 끊고 재개      (애플리케이션 레이어, L7까지 인식)
```

### 1.1 프록시 분류 한눈에 보기

```
프록시 분류
├── 방향 기준
│   ├── Forward Proxy   — 클라이언트 앞단 (내부 → 외부)
│   └── Reverse Proxy   — 서버 앞단     (외부 → 내부)
│
├── 투명성 기준
│   ├── Transparent (Intercepting) Proxy — 클라이언트가 인식 못 함
│   └── Explicit Proxy                   — 클라이언트가 프록시 주소를 설정
│
├── 프로토콜 기준
│   ├── HTTP/HTTPS Proxy   — L7, HTTP 메서드 인식
│   ├── SOCKS Proxy        — L5, TCP/UDP 범용 터널
│   └── SSL/TLS Terminating Proxy — TLS 복호화 후 처리
│
└── 역할 기준
    ├── Caching Proxy      — 응답 캐싱
    ├── Load Balancing Proxy — 부하 분산 (Reverse Proxy의 역할)
    ├── API Gateway        — 인증/라우팅/속도 제한
    └── Sidecar Proxy      — 서비스 메쉬 (Envoy, Linkerd)
```

---

## 2. Forward Proxy — 내부에서 외부로

### 2.1 동작 원리

```
[Client]  →  [Forward Proxy]  →  [Internet / 외부 서버]

① 클라이언트가 목적지 대신 프록시에게 연결 요청
② 프록시가 목적지에 대신 연결
③ 응답을 클라이언트에게 반환
```

**HTTP 요청 비교:**

```http
-- 직접 연결 --
GET /index.html HTTP/1.1
Host: example.com

-- Forward Proxy를 통한 요청 (Explicit) --
GET http://example.com/index.html HTTP/1.1    ← 전체 URL 포함
Host: example.com
Proxy-Authorization: Basic dXNlcjpwYXNz       ← 프록시 인증 헤더
```

### 2.2 HTTPS CONNECT 메서드 (터널링)

HTTPS 트래픽은 프록시가 복호화하지 않고 TCP 터널을 만든다:

```
① Client → Proxy: CONNECT example.com:443 HTTP/1.1
② Proxy → Client: HTTP/1.1 200 Connection Established
③ 이후 Client ↔ Proxy ↔ Server: 암호화된 TLS 데이터 그대로 통과
```

```bash
# curl이 CONNECT 터널을 쓰는 것 확인
curl -v --proxy http://proxy:3128 https://example.com

# 프록시 로그에서 CONNECT 메서드 확인
CONNECT example.com:443 HTTP/1.1 200   ← 성공
CONNECT example.com:443 HTTP/1.1 407   ← 인증 필요
```

**SSL Inspection (TLS 가로채기):**
기업 방화벽에서 내부 트래픽을 검사할 때:

```
Client → [MITM Proxy] → 외부 서버
          ↑
          사내 CA로 발급한 가짜 인증서 제시
          → 클라이언트가 사내 CA를 신뢰 → 복호화 가능
          → 재암호화 후 외부 서버에 전달
```

### 2.3 Forward Proxy 주요 용도

| 용도 | 설명 | 대표 도구 |
|------|------|-----------|
| 인터넷 접근 제어 | 사내 PC → 허가된 사이트만 허용 | Squid, tinyproxy |
| 익명성 | 외부 서버에 실제 IP 숨김 | Privoxy, Tor |
| 캐싱 | 자주 요청하는 외부 리소스 캐싱 | Squid |
| 로깅/감사 | 사내 인터넷 사용 기록 | Squid, Zscaler |
| 지역 우회 | 지역 제한 콘텐츠 접근 | 다양한 SaaS 제품 |

---

## 3. Reverse Proxy — 외부에서 내부로

### 3.1 동작 원리

```
[Internet]  →  [Reverse Proxy]  →  [Backend Server(s)]

클라이언트는 프록시의 존재를 모름 (백엔드 서버인 줄 앎)
프록시가 클라이언트 연결을 terminate 하고 백엔드에 새 연결
```

```
클라이언트 입장:            프록시-백엔드 입장:
Client ──TCP──→ Proxy       Proxy ──TCP──→ Backend
       ←TCP───             ←TCP────

두 개의 독립된 TCP 연결!
→ 클라이언트/백엔드 각각 다른 TCP 파라미터 적용 가능
→ 백엔드 장애 시 클라이언트 연결 유지 가능
```

### 3.2 Reverse Proxy 핵심 기능

#### 3.2.1 TLS Termination

```
Client ──HTTPS──→ [Reverse Proxy] ──HTTP──→ Backend
                   ↑
                   TLS 처리 집중화
                   - 인증서 관리 단일화
                   - 백엔드는 HTTP만 처리 (CPU 절약)
                   - OCSP Stapling, Session Resumption 적용
```

Nginx TLS 설정 예시:

```nginx
server {
    listen 443 ssl http2;
    ssl_certificate     /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    # TLS 1.2/1.3만 허용
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

    # Session Resumption (TLS 핸드쉐이크 비용 절감)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # OCSP Stapling (인증서 유효성 검사 최적화)
    ssl_stapling on;
    ssl_stapling_verify on;
}
```

#### 3.2.2 로드 밸런싱

```nginx
upstream backend {
    # Round Robin (기본)
    server backend1:8080;
    server backend2:8080;

    # Least Connections
    least_conn;
    server backend1:8080;
    server backend2:8080;

    # IP Hash (세션 고정)
    ip_hash;
    server backend1:8080;
    server backend2:8080;

    # 가중치 + 헬스체크
    server backend1:8080 weight=3;
    server backend2:8080 weight=1 backup;  # 나머지 다운 시 사용
    server backend3:8080 max_fails=3 fail_timeout=30s;
}
```

#### 3.2.3 헤더 조작

```nginx
location / {
    proxy_pass http://backend;

    # 실제 클라이언트 IP 전달
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 원래 Host 헤더 전달 (백엔드 가상 호스트 판단용)
    proxy_set_header Host $host;

    # 백엔드 응답에서 서버 정보 제거 (보안)
    proxy_hide_header Server;
    proxy_hide_header X-Powered-By;
}
```

**X-Forwarded-For 체인:**

```
Client(1.2.3.4) → Proxy1(10.0.0.1) → Proxy2(10.0.0.2) → Backend

Backend가 받는 헤더:
X-Forwarded-For: 1.2.3.4, 10.0.0.1
X-Real-IP: 1.2.3.4

주의: XFF 헤더는 클라이언트가 위조 가능
→ 신뢰할 수 있는 프록시의 IP만 화이트리스트 처리해야 함
```

#### 3.2.4 캐싱

```nginx
# 캐시 존 설정
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m
                 max_size=1g inactive=60m use_temp_path=off;

location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 302 10m;  # 200/302 응답은 10분
    proxy_cache_valid 404      1m;  # 404는 1분
    proxy_cache_use_stale error timeout updating;  # 백엔드 장애 시 stale 응답 허용

    # 캐시 상태 헤더 추가
    add_header X-Cache-Status $upstream_cache_status;
    # HIT / MISS / EXPIRED / BYPASS
}
```

---

## 4. Transparent Proxy (투명 프록시)

### 4.1 개념

클라이언트가 프록시 설정을 하지 않아도 트래픽이 자동으로 프록시를 거친다.
iptables/nftables로 패킷을 가로채 프록시로 리다이렉트한다.

```
[Client] ──────────────────────────→ [인터넷]  (클라이언트의 의도)
[Client] ──→ [iptables REDIRECT] ──→ [Proxy] ──→ [인터넷]  (실제 경로)
```

### 4.2 iptables REDIRECT 방식

```bash
# 모든 HTTP(80) 트래픽을 로컬 프록시(3128)로 리다이렉트
iptables -t nat -A PREROUTING \
    -p tcp --dport 80 \
    -j REDIRECT --to-port 3128

# HTTPS(443) 도 리다이렉트 (SSL Inspection 목적)
iptables -t nat -A PREROUTING \
    -p tcp --dport 443 \
    -j REDIRECT --to-port 3129
```

**프록시가 원래 목적지를 알아내는 방법:**

```c
// SO_ORIGINAL_DST 소켓 옵션
struct sockaddr_in orig_dst;
socklen_t len = sizeof(orig_dst);
getsockopt(client_fd, SOL_IP, SO_ORIGINAL_DST, &orig_dst, &len);
// → REDIRECT 전 원래 목적지 IP:Port 획득
```

### 4.3 TPROXY 방식 (Linux 커널 투명 프록시)

REDIRECT는 목적지 IP를 바꾸지만, TPROXY는 원래 목적지 IP/Port를 유지한다:

```bash
# TPROXY 설정
iptables -t mangle -A PREROUTING \
    -p tcp --dport 80 \
    -j TPROXY --on-port 3128 --tproxy-mark 0x1/0x1

# 마킹된 패킷을 로컬로 라우팅
ip rule add fwmark 0x1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
```

TPROXY를 쓰면 프록시 소켓에서 `getsockname()`으로 원래 목적지를 바로 얻을 수 있다.

### 4.4 Transparent Proxy 활용처

| 환경 | 용도 |
|------|------|
| 기업 게이트웨이 | 사용자 설정 없이 전체 HTTP 트래픽 검사 |
| Kubernetes CNI | Pod 트래픽을 sidecar proxy로 리다이렉트 (Istio) |
| 클라우드 NAT 게이트웨이 | 아웃바운드 트래픽 가로채기 |
| 홈라우터/UTM | 콘텐츠 필터링 |

---

## 5. SOCKS Proxy

### 5.1 SOCKS vs HTTP Proxy

| 특성 | HTTP Proxy | SOCKS Proxy |
|------|-----------|------------|
| 레이어 | L7 (HTTP 인식) | L5 (TCP/UDP 범용) |
| 프로토콜 | HTTP만 | 모든 TCP/UDP |
| 헤더 수정 | 가능 | 불가 (바이트 그대로 전달) |
| 속도 | 상대적으로 느림 | 빠름 (오버헤드 적음) |
| 용도 | 웹 트래픽 제어 | 범용 터널링 |

### 5.2 SOCKS5 핸드쉐이크 (RFC 1928)

```
① 인증 협상:
   Client → Server: VER=5, NMETHODS=1, METHODS=[0x00(무인증)]
   Server → Client: VER=5, METHOD=0x00

② 연결 요청:
   Client → Server: VER=5, CMD=1(CONNECT), RSV=0,
                    ATYP=1(IPv4), DST.ADDR=93.184.216.34, DST.PORT=80
   Server → Client: VER=5, REP=0(성공), RSV=0,
                    ATYP=1, BND.ADDR=10.0.0.1, BND.PORT=49152

③ 이후: 클라이언트 ↔ SOCKS서버 ↔ 목적지 TCP 터널
```

```bash
# curl로 SOCKS5 프록시 사용
curl --socks5-hostname proxy:1080 http://example.com

# SSH를 SOCKS5 프록시로 활용 (Dynamic Port Forwarding)
ssh -D 1080 -N user@bastion-host
# → localhost:1080이 SOCKS5 프록시가 됨
# → 브라우저에서 이 프록시 설정 시 bastion 경유

# 특정 앱에 SOCKS5 적용 (proxychains)
proxychains curl http://internal-service
```

---

## 6. 서비스 메쉬 — Sidecar Proxy (Envoy)

### 6.1 아키텍처

```
[Pod A]                              [Pod B]
┌─────────────────────────┐          ┌─────────────────────────┐
│  App Container          │          │  App Container          │
│   └─ localhost:8080     │          │   └─ localhost:8080     │
│                         │          │                         │
│  Envoy Sidecar          │          │  Envoy Sidecar          │
│   └─ 15001(outbound)    │◀────────▶│   └─ 15001(outbound)    │
│   └─ 15006(inbound)     │          │   └─ 15006(inbound)     │
└─────────────────────────┘          └─────────────────────────┘
        ↑ iptables가 모든 트래픽을 Envoy로 리다이렉트 ↑
```

**Istio가 iptables로 트래픽 가로채는 방식:**

```bash
# istio-init 컨테이너가 주입하는 iptables 규칙
iptables -t nat -N ISTIO_INBOUND
iptables -t nat -N ISTIO_REDIRECT
iptables -t nat -N ISTIO_IN_REDIRECT
iptables -t nat -N ISTIO_OUTPUT

# 인바운드: 모든 TCP 트래픽을 15006 포트로
iptables -t nat -A PREROUTING -p tcp -j ISTIO_INBOUND
iptables -t nat -A ISTIO_INBOUND -p tcp --dport 15020 -j RETURN  # Envoy 헬스체크 제외
iptables -t nat -A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT
iptables -t nat -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006

# 아웃바운드: 앱이 외부로 보내는 모든 TCP를 15001 포트로
iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT
iptables -t nat -A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN  # Envoy 자신은 제외
iptables -t nat -A ISTIO_OUTPUT -j REDIRECT --to-ports 15001
```

### 6.2 Envoy 내부 구조

```
Listener (15001)
    │
    ▼
Filter Chain
    ├── Network Filters
    │     └── HTTP Connection Manager
    │           ├── HTTP Filters
    │           │     ├── Router           ← 라우팅 결정
    │           │     ├── RBAC             ← 인가 정책
    │           │     ├── Rate Limit       ← 속도 제한
    │           │     └── JWT Authn        ← JWT 검증
    │           └── Route Config
    │                 └── Virtual Host → Route → Cluster
    │
    ▼
Cluster (업스트림 정의)
    ├── Endpoint Discovery (EDS)
    ├── Load Balancing Policy
    └── Circuit Breaker 설정
```

**xDS API — 동적 설정:**

```
[Istiod / Control Plane]
        │
        │ xDS API (gRPC 스트리밍)
        │
        ▼
[Envoy Sidecar]
  ├── LDS: Listener Discovery Service    → 어떤 포트에서 수신할지
  ├── RDS: Route Discovery Service       → 라우팅 규칙
  ├── CDS: Cluster Discovery Service     → 업스트림 클러스터 목록
  ├── EDS: Endpoint Discovery Service    → 각 클러스터의 IP:Port 목록
  └── SDS: Secret Discovery Service      → TLS 인증서/키
```

### 6.3 서비스 메쉬가 해결하는 문제

```
Without Service Mesh:               With Service Mesh (Envoy):
각 서비스가 직접 구현해야 함          Envoy가 공통으로 처리

- Retry 로직                 →       자동 재시도 + 백오프
- Circuit Breaker            →       연결 수/에러율 기반 차단
- 분산 추적                  →       자동 trace context 전파
- mTLS                       →       자동 인증서 발급/갱신
- 트래픽 분할 (Canary)        →       Weight 기반 라우팅
- 속도 제한                   →       글로벌 Rate Limiting
```

---

## 7. HAProxy 내부 동작

### 7.1 멀티프로세스 vs 멀티스레드 아키텍처

```
HAProxy 2.x (멀티스레드 모드):

Process (1개)
    ├── Thread 0 (Main)
    │     └── 설정 로드, 프로세스 관리
    ├── Thread 1 (Worker)
    │     └── epoll + Non-Blocking I/O
    ├── Thread 2 (Worker)
    │     └── epoll + Non-Blocking I/O
    └── Thread N (Worker)
          └── epoll + Non-Blocking I/O

각 스레드가 독립적으로 epoll 루프 실행
→ 락 없이 연결 처리 (Connection Locality)
→ 커널 CPU 코어 수만큼 스레드
```

### 7.2 연결 파이프라인

```
① Accept: 클라이언트 TCP SYN → SYN Queue → Accept Queue → HAProxy accept()
② Frontend: 클라이언트 연결 처리 (ACL, 인증, rate limit)
③ Backend 선택: 로드 밸런싱 알고리즘으로 서버 선택
④ 서버 연결: 커넥션 풀에서 재사용 or 신규 연결
⑤ 데이터 전달: 양방향 데이터 복사 (splice() 시스템 콜로 Zero-Copy)
⑥ 로깅/통계: 연결 종료 후 로그 기록
```

**Zero-Copy 데이터 전달 (splice):**

```c
// HAProxy 내부: read → write 대신 splice 사용
// 커널 버퍼에서 커널 버퍼로 직접 복사 → 유저공간 복사 없음
splice(pipe_read_fd, NULL, backend_fd, NULL, len, SPLICE_F_MOVE);
splice(frontend_fd, NULL, pipe_write_fd, NULL, len, SPLICE_F_MOVE);
```

### 7.3 헬스체크 메커니즘

```haproxy
backend app_servers
    option httpchk GET /health HTTP/1.1
    http-check expect status 200

    server app1 10.0.0.1:8080 check inter 2s fall 3 rise 2
    #                                 ↑      ↑        ↑    ↑
    #                             2초마다  3번실패  2번성공
    #                              체크    → DOWN  → UP

    # TCP 레벨 헬스체크 (포트만 확인)
    server db1 10.0.0.2:5432 check
```

**헬스체크 상태 전이:**

```
UP ──[fall=3 연속 실패]──→ DOWN
UP ←─[rise=2 연속 성공]─── DOWN
```

---

## 8. Nginx 내부 동작 심화

### 8.1 이벤트 루프 구조

```
Master Process
    └── fork() → Worker Process × N개 (CPU 코어 수)
                    └── epoll_wait() 루프
                          ├── 클라이언트 Accept
                          ├── 클라이언트 읽기/쓰기
                          ├── 업스트림 연결
                          └── 타이머 처리
```

```nginx
# nginx.conf 핵심 튜닝
worker_processes auto;          # CPU 코어 수만큼 Worker
worker_cpu_affinity auto;       # CPU 핀닝 (캐시 효율)

events {
    worker_connections 65536;   # Worker당 최대 연결 수
    use epoll;                  # Linux: epoll
    multi_accept on;            # accept()를 한 번에 여러 개
}
```

### 8.2 업스트림 커넥션 풀링

```nginx
upstream backend {
    server backend1:8080;
    server backend2:8080;

    # Keep-Alive 커넥션 풀
    keepalive 64;           # Worker당 최대 idle 연결 수
    keepalive_requests 1000; # 연결당 최대 요청 수
    keepalive_timeout 60s;  # idle 연결 유지 시간
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;          # keep-alive를 위해 HTTP/1.1
    proxy_set_header Connection "";   # "Connection: close" 헤더 제거
}
```

**커넥션 풀 없을 때 vs 있을 때:**

```
커넥션 풀 없음 (HTTP/1.0):
Client → Nginx → [TCP 3-way] → Backend → 요청 → 응답 → [TCP 4-way]
                 └─ 매 요청마다 TCP + TLS 핸드쉐이크 비용

커넥션 풀 있음 (Keep-Alive):
Client → Nginx → Backend (기존 연결 재사용)
                 └─ 핸드쉐이크 비용 없음, 레이턴시 대폭 감소
```

### 8.3 버퍼링 동작

```
클라이언트 ──────────→ Nginx ──────────→ Backend
    │                    │                   │
    │  느린 업로드        │  버퍼링           │
    │──────────────────→ │ (메모리/디스크)   │
    │                    │──────────────────→│
    │                    │  버퍼 채워지면     │
    │                    │  백엔드에 한번에   │
    │                    │  전송              │
```

```nginx
# 요청 버퍼링 (느린 클라이언트로부터 백엔드 보호)
proxy_request_buffering on;
client_body_buffer_size 128k;
client_max_body_size 10m;

# 응답 버퍼링 (백엔드 연결 빨리 해제)
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 24k;
```

---

## 9. API Gateway — 프록시의 고급 활용

### 9.1 API Gateway vs Reverse Proxy

```
Reverse Proxy:
- 단순 전달 + TLS + 로드밸런싱
- 경로 기반 라우팅
- 도메인 기반 라우팅

API Gateway (Reverse Proxy의 상위):
- 인증/인가 (JWT, API Key, OAuth)
- Rate Limiting (초당 요청 제한)
- Request/Response 변환
- 프로토콜 변환 (REST → gRPC)
- 회로 차단기
- 분산 추적
- API 버전 관리
```

### 9.2 Kong/Envoy 기반 Rate Limiting 내부 동작

```
Rate Limiting 알고리즘:

1. Fixed Window Counter:
   [0s─────60s] → 카운터 초기화 → [60s─────120s]
   단점: 윈도우 경계에서 2배 버스트 가능

2. Sliding Window Log:
   타임스탬프 목록 유지 → 60초 이내 요청만 카운트
   단점: 메모리 소비 큼

3. Token Bucket (가장 일반적):
   버킷에 토큰 축적 → 요청 시 토큰 소비 → 빈 버킷 = 거절
   장점: 버스트 허용 + 평균 속도 제한

4. Leaky Bucket:
   요청을 큐에 넣고 고정 속도로 처리
   장점: 완벽한 속도 평탄화, 단점: 레이턴시 증가
```

```
분산 Rate Limiting (Redis 기반):
Client → [API Gateway Node 1] ─┐
                                 ├─→ Redis (원자적 카운터)
Client → [API Gateway Node 2] ─┘

Redis INCR + EXPIRE 원자적 연산:
MULTI
INCR ratelimit:user123:1703001600  ← 현재 윈도우 키
EXPIRE ratelimit:user123:1703001600 60
EXEC
```

---

## 10. PROXY Protocol

### 10.1 문제: IP 정보 손실

```
Client(1.2.3.4) → AWS NLB → Backend
                              ↑
                         source IP = NLB IP
                         실제 클라이언트 IP를 모름
```

### 10.2 PROXY Protocol v1/v2

**v1 (텍스트):**

```
PROXY TCP4 1.2.3.4 10.0.0.1 12345 80\r\n
[이후 실제 TCP 데이터 스트림]
```

**v2 (바이너리, 더 효율적):**

```
\x0D\x0A\x0D\x0A\x00\x0D\x0A\x51\x55\x49\x54\x0A  ← 12바이트 시그니처
\x21                                                  ← 버전=2, 명령=PROXY
\x11                                                  ← 패밀리=IPv4/TCP
\x0C                                                  ← 주소 길이=12
[src_addr 4B][dst_addr 4B][src_port 2B][dst_port 2B] ← 주소 정보
[이후 실제 데이터]
```

**Nginx에서 PROXY Protocol 수신:**

```nginx
server {
    listen 80 proxy_protocol;          # PROXY Protocol 활성화
    real_ip_header proxy_protocol;
    set_real_ip_from 10.0.0.0/8;      # 신뢰할 NLB 대역

    # $proxy_protocol_addr → 실제 클라이언트 IP
    # $remote_addr → NLB IP (이걸 쓰면 안 됨)
}
```

---

## 11. 실무 장애 패턴 & 튜닝

### 11.1 Upstream Timeout 장애 분석

```
증상: 간헐적 502/504 에러

원인 체계:
                              [프록시 타임아웃 설정]
                                      │
         ┌────────────────────────────┤
         ▼                            ▼
connect_timeout (3s)       send_timeout (60s)
백엔드 연결 실패              요청 전송 중 지연

proxy_read_timeout (60s)
응답 읽기 중 지연 (DB 쿼리, 처리 시간)
```

```nginx
# Nginx 타임아웃 설정 (용도별)
proxy_connect_timeout 3s;     # 백엔드 TCP 연결 수립
proxy_send_timeout 30s;       # 요청 전송 (각 write 간격)
proxy_read_timeout 60s;       # 응답 읽기 (각 read 간격)

# 재시도 (GET 등 안전한 메서드만)
proxy_next_upstream error timeout;
proxy_next_upstream_tries 3;
proxy_next_upstream_timeout 10s;
```

### 11.2 Connection Pool 고갈

```
증상: "upstream timed out (110: Connection timed out)" 대량 발생

원인: keepalive 연결이 부족하거나 백엔드가 FIN을 갑자기 보냄

진단:
ss -s                                # 소켓 통계
netstat -an | grep ESTABLISHED | wc -l   # 현재 연결 수
cat /proc/sys/net/ipv4/ip_local_port_range  # Ephemeral 포트 범위

해결:
- keepalive 수 증가
- ip_local_port_range 확장 (1024 65535)
- TIME_WAIT 재사용 (tcp_tw_reuse=1)
```

### 11.3 헤더 크기 초과

```
증상: 400 Bad Request, "request header too large"

원인: 쿠키 누적, JWT 토큰 크기, 커스텀 헤더 과다

Nginx 기본값:
large_client_header_buffers 4 8k  → 최대 32KB

해결:
large_client_header_buffers 4 32k;
client_header_buffer_size 4k;
```

### 11.4 Sidecar Proxy 레이턴시 오버헤드

```
측정: iptables 리다이렉트 + Envoy 처리 = 보통 0.2~1ms 추가

오버헤드 발생 지점:
Client → [iptables REDIRECT] → [Envoy inbound:15006]
       → [App 처리]
       → [iptables REDIRECT] → [Envoy outbound:15001]
       → [대상 서비스 Envoy]

최적화:
- keepalive로 연결 재사용 (핸드쉐이크 제거)
- HTTP/2 멀티플렉싱
- Circuit breaker로 느린 서비스 빠른 차단
```

### 11.5 실무 진단 명령어 모음

```bash
# 프록시 연결 상태 전체 파악
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# Nginx 현재 연결 수
curl http://localhost/nginx_status
# Active connections: 847
# server accepts handled requests
#  1234567 1234567 9876543

# HAProxy 통계
echo "show info" | socat stdio /var/run/haproxy.sock

# Envoy 메트릭 (Prometheus 형식)
curl http://localhost:15090/stats/prometheus | grep upstream_cx

# 프록시 응답 헤더에서 캐시 상태 확인
curl -I https://example.com | grep -i cache

# tcpdump로 프록시 CONNECT 터널 확인
tcpdump -i eth0 -A 'tcp port 3128 and tcp[20:4] = 0x434f4e4e'
#                                                  CONN(ect)
```

---

## 12. 프록시 선택 가이드

```
요구사항별 선택:

단순 HTTP 로드밸런싱/TLS 종료
→ Nginx (설정 간단, 성능 우수)

TCP/UDP 로드밸런싱 (L4)
→ HAProxy (상태 머신 정교, 헬스체크 세밀)

서비스 메쉬 / L7 정책 동적 관리
→ Envoy (xDS API, 서킷 브레이커, 관찰가능성)

API 관리 (인증/Rate Limit/변환)
→ Kong (Nginx 기반), AWS API Gateway

포워드 프록시 / 캐싱
→ Squid (전통적), tinyproxy (경량)

사내 Zero Trust 네트워크
→ Pomerium, Boundary (ID 기반 프록시)

클라이언트 사이드 로드밸런싱 (서비스 메쉬 없이)
→ Ribbon (Netflix OSS), gRPC client-side LB
```

---

## 13. 요약 — 핵심 개념 정리

| 개념 | 핵심 포인트 |
|------|------------|
| Forward Proxy | 클라이언트 대리. CONNECT 메서드로 HTTPS 터널링 |
| Reverse Proxy | 서버 대리. TLS Terminate + LB + 캐싱 |
| Transparent Proxy | iptables REDIRECT/TPROXY로 클라이언트 모르게 가로채기 |
| SOCKS5 | L5 범용 TCP 터널. SSH 동적 포워딩이 SOCKS5 |
| Sidecar Proxy | iptables로 Pod 전체 트래픽 가로채 mTLS/추적/정책 적용 |
| PROXY Protocol | NLB 뒤에서도 클라이언트 실 IP 보존 |
| Connection Pool | Keep-Alive로 TCP 핸드쉐이크 절약 → 레이턴시 감소 |
| 버퍼링 | 느린 클라이언트로부터 백엔드 보호, 백엔드 연결 빠른 해제 |

---

## 실무 심화 포인트

1. **502 vs 504 구분**: 502(백엔드 연결 자체 실패 or 잘못된 응답), 504(백엔드 응답 타임아웃) — 원인이 다르다
2. **X-Forwarded-For 신뢰**: 가장 오른쪽 IP가 마지막으로 거친 신뢰 가능한 프록시. 왼쪽은 위조 가능
3. **PROXY Protocol 주의**: 클라이언트가 헤더를 보내지 않으면 연결 즉시 실패. 클라이언트/서버 양쪽 활성화 필수
4. **Envoy Circuit Breaker**: `max_connections`, `max_pending_requests`, `max_requests` 세 가지 임계값이 다르다
5. **Nginx upstream keepalive**: `proxy_http_version 1.1` + `proxy_set_header Connection ""` 세트로 설정해야 동작
6. **TIME_WAIT 대량 발생**: 프록시가 매 요청마다 백엔드 연결을 끊을 때 발생. keepalive로 해결
7. **SNI 기반 라우팅 (TLS Passthrough)**: TLS를 복호화하지 않고 SNI 헤더만 보고 백엔드 선택 — Nginx `stream` 모듈, HAProxy `tcp` 모드
