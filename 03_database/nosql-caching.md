# NoSQL & 캐싱

## NoSQL 분류

| 분류 | 특징 | 대표 제품 | 사용 사례 |
|------|------|---------|---------|
| Key-Value | 단순 K-V 저장, 초고속 | Redis, Memcached | 캐시, 세션, 리더보드 |
| Document | JSON 문서 저장, 유연한 스키마 | MongoDB, Firestore | 콘텐츠, 카탈로그 |
| Wide-Column | 컬럼 패밀리, 대용량 쓰기 | Cassandra, HBase | 시계열, 이벤트 |
| Graph | 노드-엣지 관계 | Neo4j, Neptune | SNS, 추천, 사기 탐지 |
| Search | 전문 검색, 분석 | Elasticsearch | 로그 분석, 검색 |

---

## Redis

### 자료구조
```bash
# String
SET key "value" EX 3600    # 1시간 TTL
GET key
INCR counter               # 원자적 증가

# Hash (객체 저장)
HSET user:1 name "Alice" age 30
HGET user:1 name
HGETALL user:1

# List (큐/스택)
RPUSH queue job1 job2      # 오른쪽에 추가
LPOP queue                 # 왼쪽에서 꺼냄
LRANGE queue 0 -1          # 전체 조회

# Set (중복 없는 집합)
SADD tags:post:1 "go" "devops"
SMEMBERS tags:post:1
SINTER tags:post:1 tags:post:2  # 교집합

# Sorted Set (점수 기반 정렬)
ZADD leaderboard 100 "Alice"
ZADD leaderboard 200 "Bob"
ZREVRANGE leaderboard 0 9 WITHSCORES  # 상위 10명

# TTL 관리
TTL key                    # 남은 시간(초)
PERSIST key               # TTL 제거
EXPIRE key 3600           # TTL 설정
```

### Redis 활용 패턴

**캐싱**
```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(cache_key, 3600, json.dumps(user))
    return user
```

**분산 락**
```bash
SET lock:resource "token" NX EX 30  # NX=없을 때만, EX=30초
# 성공(OK): 락 획득
# 실패(nil): 이미 다른 프로세스가 락 보유
```

**Rate Limiting**
```python
def is_rate_limited(user_id):
    key = f"rate:{user_id}:{int(time.time() / 60)}"  # 분 단위 윈도우
    count = redis.incr(key)
    redis.expire(key, 60)
    return count > 100  # 분당 100 요청 제한
```

### Redis 영속성
| 방식 | 설명 | 사용 사례 |
|------|------|---------|
| RDB | 주기적 스냅샷 저장 | 백업, 빠른 재시작 |
| AOF | 모든 쓰기 명령 로그 | 데이터 손실 최소화 |
| RDB+AOF | 두 방식 병행 | 프로덕션 권장 |

---

## 캐싱 전략

### Cache-Aside (Lazy Loading)
```
읽기: 캐시 미스 → DB 조회 → 캐시 저장
쓰기: DB 업데이트 → 캐시 무효화(삭제)
```
- 가장 일반적인 패턴
- Cold Start: 처음에는 캐시 미스 다수 발생

### Write-Through
```
쓰기: DB + 캐시 동시 업데이트
읽기: 캐시 우선
```
- 캐시 일관성 보장
- 쓰기 지연 증가

### Write-Behind (Write-Back)
```
쓰기: 캐시에만 먼저 쓰기 → 비동기로 DB 반영
```
- 쓰기 성능 최고
- 장애 시 데이터 손실 위험

### Read-Through
```
읽기: 캐시 미스 시 캐시가 직접 DB 조회 후 저장
```

---

## 캐시 무효화 문제

### Cache Stampede (Thunder Herd)
- 캐시 만료 시 다수의 요청이 동시에 DB로 몰림
- **해결**: Mutex Lock / Probabilistic Early Expiration / 캐시 갱신 백그라운드 처리

### Cache Penetration
- 존재하지 않는 키를 반복 요청 → DB 부하
- **해결**: Null 값도 캐시 (짧은 TTL), Bloom Filter

### Cache Avalanche
- 다수의 캐시가 동시에 만료
- **해결**: TTL에 랜덤 값 추가 `TTL = 3600 + random(0, 300)`

---

## Elasticsearch 기초

```json
// 인덱스 = 테이블, 도큐먼트 = 행, 필드 = 컬럼

// 문서 저장
PUT /logs/_doc/1
{
  "timestamp": "2024-01-01T00:00:00",
  "level": "ERROR",
  "message": "Connection refused"
}

// 검색
GET /logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "ERROR" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}
```

---

## 실무 포인트

- **Redis maxmemory-policy**: 메모리 초과 시 동작. `allkeys-lru` (캐시용), `noeviction` (데이터 손실 불가)
- **Redis Cluster vs Sentinel**: Cluster = 샤딩, Sentinel = HA(고가용성). 대용량은 Cluster
- **MongoDB 트랜잭션**: 4.0부터 멀티 도큐먼트 트랜잭션 지원. 단 성능 비용 있음
- **Elasticsearch 샤드 수**: 나중에 변경 불가. 초기 설계가 중요 (인덱스당 20~40GB가 적정)
- **캐시 히트율 모니터링**: `redis-cli info stats`의 `keyspace_hits / (keyspace_hits + keyspace_misses)`
- **메모리 단편화**: Redis `mem_fragmentation_ratio` > 1.5이면 문제 → `MEMORY PURGE`
