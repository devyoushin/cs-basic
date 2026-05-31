# 데이터베이스 내부 구조 (Senior Level)

## B-Tree 인덱스 내부

### 페이지 구조
```
B+Tree (InnoDB, PostgreSQL 모두 B+Tree 사용):
                    [Root Page]
                   /           \
          [Internal Page]   [Internal Page]
          /      |      \
    [Leaf]  [Leaf]  [Leaf] ← 실제 데이터 (또는 PK 포인터)
       ↔      ↔      ↔        ← Leaf 간 양방향 링크 (범위 스캔에 활용)
```

- **페이지 크기**: InnoDB 기본 16KB, PostgreSQL 8KB
- **Fill Factor**: 페이지를 얼마나 채울지. 삽입이 많으면 낮게 설정 (페이지 분할 감소)
- **페이지 분할 (Page Split)**: 가득 찬 페이지에 삽입 시 → 새 페이지 할당 + 데이터 재배치 → 성능 저하, 단편화

```sql
-- InnoDB 인덱스 페이지 단편화 확인
SELECT TABLE_NAME, INDEX_NAME,
       STAT_VALUE AS pages,
       STAT_VALUE * 16384 / 1024 / 1024 AS "size_MB"
FROM information_schema.INNODB_INDEX_STATS
WHERE STAT_NAME = 'size' AND TABLE_NAME = 'orders';

-- 테이블 재구성 (단편화 제거)
ALTER TABLE orders ENGINE=InnoDB;  -- Online DDL (MySQL 5.6+)
```

### 클러스터드 인덱스 vs 세컨더리 인덱스
```
InnoDB:
- 클러스터드 인덱스(PK): Leaf에 실제 행 데이터 저장
- 세컨더리 인덱스: Leaf에 PK 값만 저장 → 행 필요 시 PK로 재조회 (이중 조회)

Covering Index: 세컨더리 인덱스만으로 쿼리 완성 (PK 재조회 없음)
SELECT email FROM users WHERE status = 'active'
→ (status, email) 복합 인덱스면 covering index
```

---

## MVCC (Multi-Version Concurrency Control)

읽기와 쓰기가 서로를 블로킹하지 않는 핵심 메커니즘.

### PostgreSQL MVCC
```
각 행(tuple)에는 숨겨진 시스템 컬럼이 있음:
  xmin: 이 버전을 삽입한 트랜잭션 ID
  xmax: 이 버전을 삭제/업데이트한 트랜잭션 ID (0이면 현재 유효)
  ctid: 현재 최신 버전의 물리적 위치

UPDATE users SET name = 'Bob' WHERE id = 1:
  1. 기존 행의 xmax에 현재 XID 기록 (논리적 삭제)
  2. 새 행 삽입 (xmin = 현재 XID)
  → 두 버전이 동시에 존재!

트랜잭션의 스냅샷:
  "내 XID보다 큰 트랜잭션이 만든 버전은 안 보임"
  → 읽기는 항상 비블로킹
```

```sql
-- 행의 숨겨진 MVCC 컬럼 확인 (PostgreSQL)
SELECT ctid, xmin, xmax, * FROM users WHERE id = 1;

-- 활성 트랜잭션과 스냅샷 정보
SELECT pid, xact_start, state, query,
       age(backend_xid) as xid_age
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;

-- Dead Tuple 현황 (VACUUM 필요성 판단)
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### VACUUM의 동작
```
VACUUM이 하는 일:
1. Dead tuple의 xmax 확인 → 어떤 트랜잭션도 이 버전 안 보면 → 공간 회수
2. XID Wraparound 방지 (32bit XID, 약 21억 개)
3. Visibility Map 업데이트 (All-Visible 페이지 표시)
4. FSM (Free Space Map) 업데이트

VACUUM FULL:
- 테이블을 완전히 재작성 → 단편화 제거
- 강한 락 필요 → 프로덕션에서 주의
- pg_repack 사용 권장 (락 최소화)
```

```sql
-- XID Wraparound 위험도 확인
SELECT datname, age(datfrozenxid) as xid_age
FROM pg_database
ORDER BY xid_age DESC;
-- age > 200,000,000 이면 주의, > 1,500,000,000 이면 위험!

-- 자동 VACUUM 튜닝
ALTER TABLE hot_table SET (
  autovacuum_vacuum_scale_factor = 0.01,  -- 기본 0.2
  autovacuum_analyze_scale_factor = 0.005,
  autovacuum_vacuum_cost_delay = 2        -- ms (기본 2, 낮출수록 빠르지만 I/O 부하)
);
```

---

## InnoDB 내부

### 버퍼 풀 (Buffer Pool)
```
디스크 페이지를 메모리에 캐싱하는 InnoDB 핵심 구조.
LRU 변형 알고리즘 (Young + Old 서브리스트):

Young (5/8): 자주 접근되는 hot 페이지
Old  (3/8): 처음 로드된 페이지 → 재접근 없으면 교체
→ 풀 스캔이 버퍼 풀을 오염시키는 것 방지
```

```sql
-- 버퍼 풀 현황
SELECT POOL_ID,
       FORMAT(POOL_SIZE * 16384 / 1024 / 1024, 0) AS "Pool MB",
       FORMAT(FREE_BUFFERS * 16384 / 1024 / 1024, 0) AS "Free MB",
       ROUND((PAGES_MADE_YOUNG / PAGES_MADE_NOT_YOUNG), 2) AS young_ratio,
       HIT_RATE / 1000.0 AS hit_rate
FROM information_schema.INNODB_BUFFER_POOL_STATS;
-- hit_rate < 0.99 이면 버퍼 풀 크기 부족

-- 버퍼 풀 크기 (전체 메모리의 70~80%)
SET GLOBAL innodb_buffer_pool_size = 12 * 1024 * 1024 * 1024;  -- 12GB
```

### WAL (Write-Ahead Log) / Redo Log

```
트랜잭션 커밋 과정:
1. 변경 내용을 Buffer Pool(메모리)에 적용 (dirty page)
2. Redo Log(WAL)에 변경 기록 → 디스크 fsync
3. 커밋 완료 반환
4. 백그라운드로 dirty page를 데이터파일에 기록 (checkpoint)

장애 복구:
재시작 → Redo Log 재생 → 마지막 체크포인트 이후 변경 복원
```

```bash
# InnoDB Redo Log 설정
innodb_log_file_size = 2G          # 크게 설정 = 체크포인트 빈도 감소 = I/O 감소
innodb_log_files_in_group = 2      # 기본값
innodb_flush_log_at_trx_commit = 1 # 1=안전(fsync), 2=성능우선(OS 캐시)
# trx_commit=2 설정 시: 초당 ~1초치 트랜잭션 손실 가능하지만 성능 대폭 향상

# WAL 크기 확인 (MySQL 8.0+: 동적 조정 가능)
SHOW VARIABLES LIKE 'innodb_log%';
```

---

## 쿼리 실행 계획 심화

### PostgreSQL EXPLAIN 읽기
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id;

-- 출력 해석:
-- Seq Scan: 풀 스캔 (인덱스 없거나 선택성 낮음)
-- Index Scan: 인덱스 + 힙 접근
-- Index Only Scan: 인덱스만으로 처리 (covering index)
-- Bitmap Heap Scan: 인덱스로 위치 수집 후 배치로 힙 접근 (랜덤 I/O 줄임)
-- Hash Join: 작은 쪽을 해시테이블로 만들어 조인
-- Nested Loop: 한쪽이 작고 다른 쪽에 인덱스 있을 때
-- Merge Join: 양쪽 정렬되어 있을 때

-- Buffers: hit=X read=Y
-- hit: 버퍼 캐시 히트 (빠름)
-- read: 디스크 읽기 (느림)
-- read 가 크면 cache miss → 버퍼 부족 또는 콜드 캐시
```

### 플래너 통계 업데이트
```sql
-- 통계가 오래되면 플래너가 잘못된 실행 계획 선택
ANALYZE users;  -- 통계 수집

-- 통계 타겟 높이기 (기본 100, 복잡한 컬럼에 올림)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;

-- 플래너 비용 파라미터 조정 (SSD 환경)
SET random_page_cost = 1.1;  -- 기본 4.0 (HDD 기준)
SET seq_page_cost = 1.0;
-- SSD에서 랜덤/순차 I/O 차이가 작음 → 인덱스 스캔 더 선호하게 됨
```

---

## 복제 (Replication) 내부

### PostgreSQL WAL 스트리밍 복제
```
Primary: WAL 생성 → WAL Sender 프로세스
Replica: WAL Receiver → 수신 → WAL 재생

복제 지연 모니터링:
SELECT client_addr,
       state,
       sent_lsn - write_lsn  AS "write_lag_bytes",
       sent_lsn - flush_lsn  AS "flush_lag_bytes",
       sent_lsn - replay_lsn AS "replay_lag_bytes",
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

### MySQL GTID 복제
```sql
-- GTID (Global Transaction ID): server_uuid:transaction_id
-- 복제 위치를 binlog 파일+오프셋 대신 GTID로 추적

-- 복제 상태
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: 복제 지연 (초)
-- Retrieved_Gtid_Set vs Executed_Gtid_Set: 수신 vs 적용된 GTID

-- 복제 지연 원인 분석
-- 1. 대용량 트랜잭션 (DDL, 대량 업데이트)
-- 2. 레플리카의 병렬 처리 스레드 부족
SET GLOBAL replica_parallel_workers = 8;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
```

---

## 락 경합 심화 분석

```sql
-- PostgreSQL: 현재 대기 중인 락
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE NOT blocked.granted;

-- InnoDB: 락 대기 현황
SELECT r.trx_id waiting_id,
       r.trx_query waiting_query,
       b.trx_id blocking_id,
       b.trx_query blocking_query,
       TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) wait_sec
FROM information_schema.INNODB_TRX b
JOIN information_schema.INNODB_TRX r
  ON r.trx_id != b.trx_id
WHERE r.trx_wait_started IS NOT NULL
ORDER BY wait_sec DESC;

-- 데드락 로그 확인 (InnoDB)
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션 확인
```

---

## 실무 심화 포인트

- **Hot Page Contention**: 단조 증가 PK(AUTO_INCREMENT)에 동시 삽입 → B-Tree 오른쪽 끝 페이지만 업데이트 → 락 경합. UUID v4로 분산하면 오히려 랜덤 삽입 → 캐시 미스. UUID v7(시간 정렬) 또는 Snowflake ID가 절충안
- **Connection Pooling 깊이**: PgBouncer의 transaction mode vs session mode. transaction mode에서는 `SET`, prepared statement 주의
- **Parallel Query**: PostgreSQL `max_parallel_workers_per_gather`. OLAP 쿼리에서 활용, 하지만 OLTP에서는 오히려 독이 될 수 있음
- **Write Amplification**: SSD에서 작은 랜덤 쓰기 → 페이지 전체 재기록 → InnoDB double write buffer가 이를 완화
- **FSM (Free Space Map) 고갈**: PostgreSQL에서 massive DELETE 후 VACUUM 안 하면 테이블이 계속 커짐. Bloat 모니터링 필수
- **MySQL 8.0 Redo Log 동적 조정**: `innodb_redo_log_capacity`로 온라인 변경 가능
