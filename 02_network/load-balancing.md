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

| 알고리즘 | 설명 | 적합한 경우 |
|---------|------|-----------|
| Round Robin | 순서대로 분배 | 서버 스펙 동일 |
| Weighted Round Robin | 가중치 비례 분배 | 서버 스펙 다를 때 |
| Least Connections | 연결 수 가장 적은 서버 | 요청 처리 시간이 다양할 때 |
| IP Hash | 클라이언트 IP 기반 | 세션 고정(Sticky Session) |
| Random | 무작위 | 단순 분산 |
| Least Response Time | 응답시간 가장 짧은 서버 | 성능 중심 |

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
