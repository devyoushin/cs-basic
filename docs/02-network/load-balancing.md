# 로드 밸런싱 (Load Balancing)

## L4 vs L7 로드밸런서

| 구분 | L4 (Transport) | L7 (Application) |
|------|--------------|-----------------|
| 기준 | IP + Port | HTTP 헤더, URL, 쿠키 |
| 속도 | 빠름 (패킷 레벨) | 느림 (컨텐츠 분석) |
| 기능 | 포트 기반 라우팅 | URL 기반 라우팅, SSL Termination |
| 예시 | AWS NLB, HAProxy TCP | AWS ALB, nginx, Traefik |
| 세션 | TCP 연결 기반 | HTTP 세션/쿠키 기반 |

---

## 알고리즘

### Round Robin (라운드 로빈)
```
요청 순서: 1  2  3  4  5  6  7  8
배분 서버: A  B  C  A  B  C  A  B
```
- 순서대로 순환 배분. 가장 단순하고 균등
- **적합**: 서버 스펙 동일, 요청 처리 시간 비슷한 경우
- **단점**: 서버 성능이 다를 때 성능 좋은 서버에 과부하 발생

### Weighted Round Robin (가중치 라운드 로빈)
```
서버 A (weight=3), 서버 B (weight=1)
배분: A A A B A A A B ...
```
- 서버별 가중치 비례로 요청 분배
- **적합**: 서버 사양이 다를 때 (고사양 서버에 더 많이 배분)

### Least Connections (최소 연결)
```
서버 A: 현재 연결 10개
서버 B: 현재 연결 3개  ← 다음 요청은 B로
서버 C: 현재 연결 7개
```
- 현재 **활성 연결 수**가 가장 적은 서버로 라우팅
- **적합**: 요청마다 처리 시간이 크게 다를 때 (파일 업로드 + API 혼합)
- **단점**: 연결 수 추적 오버헤드, 연결은 적어도 무거운 요청일 수 있음

### Weighted Least Connections
```
서버 A (weight=2, conn=4): 점수 = 4/2 = 2
서버 B (weight=1, conn=3): 점수 = 3/1 = 3  ← 점수 낮은 A로
```
- 연결 수를 가중치로 나눈 값이 가장 낮은 서버 선택
- 사양 다른 서버에서 Least Connections의 개선판

### IP Hash (IP 해시)
```
client IP → hash() → 특정 서버에 항상 매핑
203.0.113.1 → hash → 서버 A (항상)
203.0.113.2 → hash → 서버 B (항상)
```
- 같은 클라이언트 IP는 항상 같은 서버로 → **Sticky Session** 효과
- **적합**: 서버 측 세션 있을 때, 파일 업로드 멀티파트 요청
- **단점**: 특정 IP 집중 시 불균등. 서버 추가/제거 시 매핑 변경됨

### Least Response Time (최소 응답 시간)
```
서버 A: 평균 응답 50ms
서버 B: 평균 응답 20ms  ← 다음 요청은 B로
서버 C: 평균 응답 35ms
```
- 응답 시간 + 활성 연결 수 조합으로 최적 서버 선택
- **적합**: 서버 간 성능 편차 클 때, 레이턴시 민감한 서비스

### 알고리즘 선택 가이드

| 상황 | 권장 알고리즘 |
|------|------------|
| 서버 스펙 동일, 단순 분산 | Round Robin |
| 서버 스펙 다름 | Weighted Round Robin |
| 처리 시간 불균등 | Least Connections |
| 세션 유지 필요 | IP Hash |
| 레이턴시 최소화 | Least Response Time |

```nginx
# Nginx upstream 알고리즘 설정
upstream backend {
    # Round Robin (기본값, 별도 지시어 없음)
    server app1:8080;
    server app2:8080;
}

upstream backend_weighted {
    server app1:8080 weight=3;  # Weighted Round Robin
    server app2:8080 weight=1;
}

upstream backend_lc {
    least_conn;                 # Least Connections
    server app1:8080;
    server app2:8080;
}

upstream backend_hash {
    ip_hash;                    # IP Hash
    server app1:8080;
    server app2:8080;
}
```

---

## Health Check

```nginx
# nginx upstream health check
upstream backend {
    server app1:8080;
    server app2:8080;

    # passive health check (기본)
    # 요청 실패 시 일시적으로 제외
}
```

```yaml
# AWS ALB Health Check
HealthCheckProtocol: HTTP
HealthCheckPath: /health
HealthCheckIntervalSeconds: 30
HealthyThresholdCount: 2
UnhealthyThresholdCount: 3
HealthCheckTimeoutSeconds: 5
```

### 헬스체크 엔드포인트 설계
```json
GET /health
{
  "status": "healthy",
  "version": "1.2.3",
  "checks": {
    "database": "ok",
    "redis": "ok"
  }
}
```
- **Liveness**: 프로세스 살아있는지 (단순 200 OK)
- **Readiness**: 트래픽 받을 준비됐는지 (DB 연결 등 확인)

---

## SSL/TLS Termination

```
Client ──[HTTPS]──→ LB ──[HTTP]──→ Backend

장점:
- 백엔드 서버의 TLS 처리 부담 감소
- 인증서를 한 곳에서 관리
- 백엔드 트래픽 분석 용이

주의:
- 내부 네트워크도 암호화가 필요하면 Re-encryption 또는 Pass-through
```

---

## Sticky Session (Session Affinity)

동일 클라이언트가 동일 서버로 라우팅:

```nginx
upstream backend {
    ip_hash;  # IP 기반 고정
    server app1:8080;
    server app2:8080;
}
```

**문제점**: 특정 서버에 부하 집중, 서버 다운 시 세션 손실
**해결책**: 세션을 Redis 같은 외부 저장소에 저장 (Stateless 아키텍처)

---

## 글로벌 로드밸런싱 (GSLB)

- DNS 기반으로 지역별 라우팅
- **방법**: Latency 기반, 지리적 기반, 가중치 기반
- **예시**: AWS Route 53 Routing Policy, Cloudflare Load Balancing

```
사용자 (서울) → DNS 조회 → 서울 리전 서버
사용자 (뉴욕) → DNS 조회 → 버지니아 리전 서버
```

---

## 연결 제한 & Rate Limiting

```nginx
# nginx rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
    }
}
```

---

## 실무 포인트

- **Connection Draining (Deregistration Delay)**: 인스턴스 제거 전 기존 연결 처리 완료 대기 (AWS ALB 기본 300초)
- **Pre-warming**: 트래픽 급증 예상 시 ALB 사전 확장 요청 (AWS Support)
- **X-Forwarded-For 신뢰**: 로드밸런서 IP만 신뢰하도록 설정 (IP 스푸핑 방지)
- **Keep-Alive 설정 충돌**: nginx의 `keepalive_timeout`이 ALB 유휴 타임아웃(60초)보다 짧으면 502 발생. nginx를 75초로 설정 권장
- **WebSocket 지원**: HTTP Upgrade 헤더 처리 필요. nginx `proxy_http_version 1.1` + `Upgrade/Connection` 헤더 설정
- **그라파나 대시보드**: Target response time, healthy host count, 5xx 비율을 핵심 지표로 모니터링
