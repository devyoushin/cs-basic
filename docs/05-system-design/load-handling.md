# DB/Web 부하 대응 전략

## 개요

서비스 부하 대응은 시스템의 병목 지점을 식별하고, 장애 확산을 막으면서, 처리 용량을 단계적으로 늘리는 작업이다.

부하는 크게 데이터베이스 (Database) 부하와 웹 계층 (Web Tier) 부하로 나눌 수 있다.

- **DB 부하**: 쿼리 지연, 락 대기, 커넥션 고갈, 디스크 I/O 병목, 복제 지연으로 나타난다.
- **Web 부하**: CPU 사용률 증가, 메모리 부족, 스레드/이벤트 루프 포화, 커넥션 큐 증가, 5xx 오류 증가로 나타난다.

핵심은 "서버를 늘린다"가 아니라 **어디가 병목인지 먼저 확인하고, 즉시 완화책과 구조적 개선책을 구분해 적용하는 것**이다.

---

## 공통 진단 원칙

### USE 방법론

리소스 단위로 사용률 (Utilization), 포화도 (Saturation), 오류 (Errors)를 확인한다.

| 항목 | 확인 대상 | 예시 |
|------|----------|------|
| 사용률 | CPU, Memory, Disk, Network | CPU 90%, Disk util 95% |
| 포화도 | 대기열, 큐 길이, 대기 시간 | Load Average 증가, DB lock wait 증가 |
| 오류 | 실패 요청, 타임아웃, 재시도 | 5xx, connection timeout |

### RED 방법론

서비스 단위로 요청량 (Rate), 오류율 (Errors), 지연시간 (Duration)을 확인한다.

| 항목 | 의미 |
|------|------|
| Rate | 초당 요청 수, QPS, RPS |
| Errors | 4xx/5xx, DB 에러, 타임아웃 |
| Duration | p50, p95, p99 레이턴시 |

### 우선 확인 지표

```bash
# Linux 서버 리소스 확인
top
vmstat 1
iostat -x 1
sar -n DEV 1

# 포트/커넥션 상태 확인
ss -s
ss -tan state established
ss -tan state time-wait
```

| 증상 | 가능성 높은 병목 |
|------|----------------|
| CPU 90% 이상, Load Average 높음 | 애플리케이션 연산, 쿼리 CPU 비용, GC |
| 메모리 부족, Swap 사용 | 캐시 과다, 메모리 누수, JVM Heap 부족 |
| Disk await 증가 | DB I/O 병목, 로그/스왑 I/O |
| Network retransmit 증가 | 네트워크 혼잡, 패킷 손실 |
| DB connection timeout | 커넥션 풀 고갈, DB 처리 지연 |
| p99만 급증 | 일부 느린 쿼리, 락, 외부 API 지연 |

---

## DB 부하 대응

### 1. 즉시 완화

DB가 이미 포화된 상황에서는 새로운 부하를 줄이는 것이 우선이다.

- **트래픽 제한 (Rate Limiting)**: 비핵심 API, 검색 API, 대량 조회 API에 요청 제한을 적용한다.
- **기능 임시 비활성화 (Feature Flag)**: 추천, 통계, 대시보드처럼 DB를 많이 쓰는 부가 기능을 끈다.
- **캐시 TTL 연장**: Redis, CDN, 애플리케이션 캐시의 만료 시간을 늘려 DB 읽기를 줄인다.
- **배치/크론 중단**: 리포트 생성, 정산, 대량 동기화 작업을 일시 중지한다.
- **읽기 트래픽 분리**: 가능한 조회 요청을 읽기 복제본 (Read Replica)으로 보낸다.
- **DB 커넥션 상한 조정**: 애플리케이션 인스턴스 증가가 DB 커넥션 폭증으로 이어지지 않도록 풀 크기를 제한한다.

주의할 점은 커넥션 풀을 무작정 늘리면 DB가 더 빨리 포화된다는 것이다. 커넥션은 처리량을 보장하는 장치가 아니라 동시 실행량을 제한하는 장치다.

### 2. 원인 진단

#### 느린 쿼리 확인

```sql
-- MySQL: 실행 중인 쿼리 확인
SHOW FULL PROCESSLIST;

-- PostgreSQL: 오래 실행 중인 쿼리 확인
SELECT pid,
       now() - query_start AS duration,
       state,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

#### 실행 계획 확인

```sql
-- MySQL / PostgreSQL 공통
EXPLAIN SELECT *
FROM orders
WHERE user_id = 100
ORDER BY created_at DESC
LIMIT 20;
```

확인할 내용은 다음과 같다.

| 항목 | 문제 신호 |
|------|----------|
| Full Scan | 인덱스 없이 전체 테이블 스캔 |
| Sort | 큰 결과 집합을 정렬 |
| Join 방식 | 큰 테이블끼리 비효율적으로 조인 |
| Rows | 예상 스캔 행 수가 과도함 |
| Using temporary | 임시 테이블 생성 |

#### 락 대기 확인

```sql
-- PostgreSQL: 락 대기 중인 쿼리
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks
  ON blocked_locks.pid = blocked.pid
JOIN pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
 AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
 AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
 AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
 AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
 AND blocking_locks.pid <> blocked_locks.pid
JOIN pg_stat_activity blocking
  ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

락 대기가 길면 쿼리 튜닝보다 먼저 긴 트랜잭션, 대량 UPDATE/DELETE, 배치 작업을 확인한다.

### 3. 단기 개선

#### 인덱스 추가

인덱스는 WHERE, JOIN, ORDER BY 조건을 기준으로 설계한다.

```sql
-- 사용자별 최신 주문 조회 최적화
CREATE INDEX idx_orders_user_created_at
ON orders (user_id, created_at DESC);
```

복합 인덱스 (Composite Index)는 선두 컬럼 순서가 중요하다. `WHERE user_id = ? ORDER BY created_at DESC`라면 `(user_id, created_at)` 순서가 자연스럽다.

#### 쿼리 범위 축소

```sql
-- 나쁜 예: 불필요한 컬럼과 큰 범위 조회
SELECT *
FROM access_logs
WHERE created_at >= '2026-01-01';

-- 좋은 예: 필요한 컬럼과 제한된 범위 조회
SELECT id, user_id, path, created_at
FROM access_logs
WHERE created_at >= now() - interval '1 day'
ORDER BY created_at DESC
LIMIT 100;
```

#### 페이지네이션 개선

OFFSET이 커질수록 DB는 건너뛸 행도 읽어야 한다. 큰 목록에는 커서 기반 페이지네이션 (Cursor-based Pagination)을 사용한다.

```sql
SELECT id, title, created_at
FROM posts
WHERE created_at < :last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;
```

#### 캐시 적용

| 패턴 | 설명 | 적합한 경우 |
|------|------|------------|
| Cache-Aside | 앱이 캐시 조회 후 미스 시 DB 조회 | 일반적인 조회 API |
| Write-Through | 쓰기 시 캐시와 DB를 함께 갱신 | 강한 최신성이 필요한 데이터 |
| TTL Cache | 일정 시간 동안 결과 재사용 | 통계, 랭킹, 설정 |

캐시는 DB 부하를 줄이지만 캐시 스탬피드 (Cache Stampede)를 만들 수 있다. 인기 키가 동시에 만료되지 않도록 TTL에 지터 (Jitter)를 섞는다.

### 4. 구조적 개선

#### 읽기 복제본

```text
Write API  → Primary DB
Read API   → Read Replica 1
Read API   → Read Replica 2
```

읽기 복제본은 읽기 부하를 분산하지만 복제 지연 (Replication Lag)이 발생할 수 있다. 방금 쓴 데이터를 바로 읽어야 하는 요청은 Primary DB를 사용하거나 Read-after-write 정책을 둔다.

#### 파티셔닝

```sql
-- PostgreSQL 예시: 월 단위 Range Partition
CREATE TABLE access_logs (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    path TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);
```

파티셔닝 (Partitioning)은 큰 테이블을 논리적으로 나눠 스캔 범위를 줄인다. 로그, 이벤트, 주문 이력처럼 시간 조건이 많은 테이블에 적합하다.

#### 샤딩

```text
user_id % 4 = 0 → Shard 0
user_id % 4 = 1 → Shard 1
user_id % 4 = 2 → Shard 2
user_id % 4 = 3 → Shard 3
```

샤딩 (Sharding)은 데이터를 여러 DB에 수평 분할한다. 처리 용량은 커지지만 Cross-shard Join, 글로벌 트랜잭션, 리밸런싱이 어려워진다. 인덱스/캐시/복제본/파티셔닝으로 해결되지 않을 때 검토한다.

#### 비동기 처리

```text
Client → API → Queue → Worker → DB
         ↓
       즉시 응답
```

DB 쓰기 작업 중 즉시 완료될 필요가 없는 작업은 큐 (Queue)로 넘긴다. 이메일, 알림, 통계 집계, 외부 API 동기화가 대표적이다.

---

## Web 부하 대응

### 1. 즉시 완화

웹 계층이 포화된 상황에서는 요청을 빠르게 실패시키거나, 캐시 가능한 요청을 애플리케이션까지 보내지 않는 것이 중요하다.

- **오토스케일링 (Auto Scaling)**: CPU, RPS, p95 레이턴시, 큐 길이를 기준으로 인스턴스를 늘린다.
- **로드밸런서 헬스체크 강화**: 비정상 인스턴스를 빠르게 제외한다.
- **정적 리소스 CDN 캐싱**: JS, CSS, 이미지 요청을 Origin까지 보내지 않는다.
- **API 응답 캐싱**: 변경이 적은 조회 API에 `Cache-Control`, Reverse Proxy Cache, Redis Cache를 적용한다.
- **Rate Limiting**: 사용자/IP/API Key 단위로 과도한 요청을 제한한다.
- **Timeout 단축**: 느린 외부 의존성 때문에 웹 스레드가 묶이지 않도록 한다.
- **Graceful Degradation**: 핵심 기능만 유지하고 부가 기능은 임시로 비활성화한다.

### 2. 원인 진단

#### 서버 리소스 확인

```bash
# CPU, 메모리, Load Average
top
vmstat 1

# 디스크 I/O
iostat -x 1

# 네트워크 처리량
sar -n DEV 1

# 소켓 상태 요약
ss -s
```

#### 애플리케이션 지표 확인

| 지표 | 의미 |
|------|------|
| RPS | 요청 유입량 |
| p95/p99 Latency | 사용자 체감 지연 |
| 5xx Rate | 서버 처리 실패 |
| Thread Pool Active | 작업 스레드 포화 여부 |
| Queue Length | 요청 대기열 증가 여부 |
| GC Pause | JVM 기반 서비스의 중단 시간 |
| Connection Pool Active | DB/Redis/외부 API 커넥션 고갈 |

#### 병목별 증상

| 증상 | 가능성 높은 원인 | 대응 |
|------|----------------|------|
| CPU 높음 | JSON 직렬화, 암호화, 압축, 비효율 코드 | 프로파일링, 캐시, 알고리즘 개선 |
| 메모리 증가 | 메모리 누수, 큰 응답, 과도한 캐시 | Heap Dump, 응답 크기 제한 |
| 스레드 포화 | 느린 DB/외부 API 호출 | Timeout, Bulkhead, 비동기화 |
| 502/503 증가 | 인스턴스 다운, LB 타임아웃 | 헬스체크, 인스턴스 증설 |
| 504 증가 | 업스트림 처리 지연 | DB/API 병목 확인, Timeout 조정 |
| TIME_WAIT 많음 | 짧은 연결 반복 | Keep-Alive, 커넥션 풀 |

### 3. 단기 개선

#### 수평 확장

웹 서버는 상태를 갖지 않는 Stateless 구조일수록 수평 확장이 쉽다.

```text
Client → Load Balancer → Web 1
                      → Web 2
                      → Web 3
```

세션을 로컬 메모리에 저장하면 특정 서버에 사용자가 묶인다. 세션은 Redis, DB, JWT 등 외부 저장소 또는 토큰 기반으로 분리한다.

#### 커넥션 풀 조정

```yaml
# 예시: 애플리케이션 DB 커넥션 풀
datasource:
  hikari:
    maximum-pool-size: 20
    minimum-idle: 5
    connection-timeout: 3000
```

전체 DB 커넥션 수는 다음처럼 계산한다.

```text
전체 커넥션 = 인스턴스 수 × 인스턴스당 커넥션 풀 크기
```

오토스케일링으로 웹 인스턴스가 늘어나면 DB 커넥션도 같이 늘어난다. DB의 `max_connections`보다 작게 유지하고, DB가 감당할 수 있는 동시 쿼리 수를 기준으로 잡는다.

#### Timeout과 Retry 제한

```yaml
http-client:
  connect-timeout-ms: 1000
  read-timeout-ms: 2000
  max-retries: 2
```

재시도 (Retry)는 장애를 완화할 수 있지만, 포화 상태에서는 부하를 증폭한다. Exponential Backoff와 Jitter를 사용하고, 전체 요청 데드라인을 넘지 않도록 제한한다.

#### 정적/동적 캐싱

```http
Cache-Control: public, max-age=31536000, immutable
```

정적 파일은 파일명에 해시를 포함해 장기 캐싱한다. 동적 API는 사용자별 응답인지, 권한에 따라 달라지는지 확인한 뒤 캐시 키를 설계한다.

### 4. 구조적 개선

#### 비동기 아키텍처

긴 작업은 요청-응답 경로에서 분리한다.

```text
Client → Web API → Queue → Worker
         ↓
      202 Accepted
```

사용자는 작업 ID를 받고, 상태 조회 API로 진행 상황을 확인한다. 이미지 변환, 대용량 파일 처리, 리포트 생성에 적합하다.

#### Bulkhead 패턴

```text
핵심 API       → Thread Pool A → DB
부가 기능 API  → Thread Pool B → 외부 API
관리자 API     → Thread Pool C → DB
```

격벽 (Bulkhead)은 장애가 한 영역에서 전체 서비스로 퍼지지 않도록 리소스를 분리하는 패턴이다. 외부 API가 느려져도 핵심 API의 스레드가 고갈되지 않게 만든다.

#### Circuit Breaker

```text
CLOSED → 실패율 증가 → OPEN
OPEN → 일정 시간 차단 → HALF-OPEN
HALF-OPEN → 성공 시 CLOSED, 실패 시 OPEN
```

서킷 브레이커 (Circuit Breaker)는 실패가 반복되는 의존성 호출을 잠시 차단해 웹 서버의 스레드와 커넥션을 보호한다.

#### 큐 기반 백프레셔

백프레셔 (Backpressure)는 처리 가능한 만큼만 요청을 받는 제어 방식이다. 큐 길이가 임계치를 넘으면 새 요청을 거절하거나 낮은 우선순위 작업을 버린다.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 10
```

무한 큐는 장애를 늦게 드러낼 뿐이다. 큐에는 최대 길이와 대기 시간을 둔다.

---

## DB 부하와 Web 부하 구분

| 질문 | DB 병목 가능성 | Web 병목 가능성 |
|------|---------------|----------------|
| DB CPU/I/O가 높은가 | 높음 | 낮음 |
| DB 커넥션 풀이 꽉 찼는가 | 높음 | 중간 |
| 웹 CPU만 높은가 | 낮음 | 높음 |
| 웹 스레드가 WAITING인가 | 중간 | 높음 |
| DB 쿼리 p95가 높은가 | 높음 | 낮음 |
| 정적 파일 요청이 많은가 | 낮음 | 높음 |
| 외부 API 지연이 있는가 | 낮음 | 높음 |
| 락 대기가 증가했는가 | 높음 | 낮음 |

실무에서는 둘이 함께 발생하는 경우가 많다. 예를 들어 느린 DB 쿼리는 웹 스레드를 오래 점유하고, 웹 인스턴스 증설은 DB 커넥션을 늘려 DB를 더 압박할 수 있다.

---

## 장애 대응 순서

### 1단계: 보호

- Rate Limiting 적용
- 비핵심 기능 비활성화
- 배치/크론 중단
- 캐시 TTL 연장
- 느린 의존성 Timeout 단축

### 2단계: 병목 확인

- RED 지표로 어떤 API가 느린지 확인
- USE 지표로 CPU/Memory/Disk/Network 병목 확인
- DB Slow Query, Lock Wait, Connection Pool 확인
- 웹 Thread Dump, GC, Queue Length 확인

### 3단계: 용량 확장

- Web은 수평 확장 우선
- DB는 읽기 복제본, 스펙 증설, 스토리지 IOPS 확장 검토
- 커넥션 풀 총량이 DB 한계를 넘지 않는지 확인

### 4단계: 원인 제거

- 느린 쿼리 튜닝
- 인덱스 추가
- 캐시 설계
- 동기 작업 비동기화
- 파티셔닝/샤딩 검토
- 외부 의존성에 Circuit Breaker 적용

### 5단계: 재발 방지

- SLO와 알람 임계치 조정
- 부하 테스트 (Load Test) 수행
- 런북 (Runbook) 업데이트
- 병목 지표 대시보드 추가
- 배치 작업 시간대와 동시성 제한 조정

---

## 면접 포인트

| 질문 | 핵심 답변 |
|------|----------|
| Q: DB 부하가 높을 때 가장 먼저 무엇을 확인하는가? | A: Slow Query, Lock Wait, Connection Pool, CPU/I/O 지표를 확인해 읽기/쓰기/락/리소스 병목을 구분한다. |
| Q: DB 커넥션 풀을 늘리면 성능이 좋아지는가? | A: 항상 그렇지 않다. DB가 이미 포화 상태라면 커넥션 증가는 동시 쿼리 수를 늘려 병목을 악화시킨다. |
| Q: Web 서버 부하는 어떻게 해결하는가? | A: Stateless 구조라면 수평 확장이 기본이고, 캐시, Rate Limiting, Timeout, 비동기 처리로 요청 경로의 부하를 줄인다. |
| Q: Web 인스턴스를 늘렸는데 장애가 심해질 수 있는 이유는 무엇인가? | A: 인스턴스마다 DB 커넥션 풀이 생기므로 전체 DB 커넥션과 쿼리 동시성이 증가해 DB를 더 압박할 수 있다. |
| Q: 캐시 적용 시 주의할 점은 무엇인가? | A: 데이터 정합성, 캐시 무효화, 캐시 스탬피드, 사용자별 권한이 반영된 캐시 키 설계를 주의해야 한다. |
| Q: 504 Gateway Timeout이 증가하면 어디를 봐야 하는가? | A: 웹 서버 자체보다 업스트림인 애플리케이션, DB, 외부 API의 처리 지연과 Timeout 설정을 함께 확인한다. |

---

## 실무 포인트

- **먼저 줄이고 나중에 늘린다**: 포화 상태에서는 요청량, 배치, 비핵심 기능을 줄이는 것이 서버 증설보다 빠를 수 있다.
- **Web Scale-out은 DB 보호와 함께 한다**: 웹 인스턴스를 늘릴 때 DB 커넥션 총량과 쿼리 동시성을 반드시 계산한다.
- **p99를 본다**: 평균 레이턴시는 일부 사용자의 심각한 지연을 숨긴다.
- **캐시는 장애 대응 도구이자 장애 원인이다**: 캐시 미스 폭증, 동시 만료, 잘못된 캐시 키가 DB 장애를 만들 수 있다.
- **타임아웃 없는 호출은 장애 전파 경로다**: DB, Redis, 외부 API, 내부 API 호출에는 명시적 Timeout을 둔다.
- **무한 재시도 금지**: Retry는 반드시 횟수 제한, Backoff, Jitter, 전체 Deadline을 가져야 한다.
- **런북으로 자동화한다**: 자주 쓰는 진단 쿼리, 로그 검색, 대시보드 링크, 롤백 절차를 문서화한다.

---

## TIP

- DB 부하가 높을 때는 `SELECT *`, 큰 `OFFSET`, 인덱스 없는 `ORDER BY`, 대량 `UPDATE/DELETE`, 긴 트랜잭션을 우선 의심한다.
- Web 부하가 높을 때는 CPU-bound인지 I/O-bound인지 먼저 나눈다.
- 웹 서버 10대, 커넥션 풀 30개면 DB에는 최대 300개 커넥션이 생길 수 있다.
- 장애 중에는 완벽한 최적화보다 빠른 완화가 우선이다. Rate Limiting, Feature Flag, 캐시 TTL 조정은 효과가 빠르다.
- 장애 후에는 부하 테스트로 같은 상황을 재현하고, 알람이 장애 전에 울렸는지 확인한다.
