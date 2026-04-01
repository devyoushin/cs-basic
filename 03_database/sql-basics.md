# 데이터베이스 기초 (SQL)

## ACID 속성

| 속성 | 설명 | 예시 |
|------|------|------|
| **A**tomicity | 트랜잭션은 전부 성공하거나 전부 실패 | 계좌이체: 출금+입금이 동시에 성공/실패 |
| **C**onsistency | 트랜잭션 후 데이터가 항상 일관된 상태 | 잔액이 음수가 될 수 없음 |
| **I**solation | 동시 트랜잭션이 서로 영향 없음 | 다른 트랜잭션의 중간 상태 안 보임 |
| **D**urability | 커밋된 트랜잭션은 영구 저장 | 서버 재시작해도 데이터 유지 |

---

## 트랜잭션 격리 수준

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|-----------|--------------------|-----------|
| READ UNCOMMITTED | 가능 | 가능 | 가능 |
| READ COMMITTED | 불가 | 가능 | 가능 |
| REPEATABLE READ | 불가 | 불가 | 가능 (InnoDB는 불가) |
| SERIALIZABLE | 불가 | 불가 | 불가 |

- **Dirty Read**: 커밋 안 된 데이터 읽기
- **Non-Repeatable Read**: 같은 쿼리를 두 번 실행하면 결과가 다름 (중간에 UPDATE)
- **Phantom Read**: 같은 쿼리를 두 번 실행하면 행 수가 다름 (중간에 INSERT/DELETE)

MySQL InnoDB 기본: **REPEATABLE READ**
PostgreSQL 기본: **READ COMMITTED**

---

## 인덱스

### B-Tree 인덱스 (기본)
```sql
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_date ON orders(user_id, created_at);  -- 복합 인덱스
```

### 인덱스가 사용되는 경우
```sql
WHERE email = 'foo@example.com'      -- 동등 비교
WHERE created_at > '2024-01-01'      -- 범위 검색
ORDER BY last_name                    -- 정렬
JOIN users ON orders.user_id = users.id  -- 조인
```

### 인덱스가 사용 안 되는 경우
```sql
WHERE YEAR(created_at) = 2024        -- 함수 적용
WHERE email LIKE '%gmail.com'        -- 앞에 % 와일드카드
WHERE status != 'active'             -- 부정 연산 (데이터에 따라 다름)
```

### 복합 인덱스 주의사항
```sql
-- 인덱스: (a, b, c)
WHERE a = 1                    -- O 사용
WHERE a = 1 AND b = 2          -- O 사용
WHERE a = 1 AND b = 2 AND c = 3  -- O 사용
WHERE b = 2                    -- X 미사용 (앞 컬럼 없음)
WHERE a = 1 AND c = 3          -- a만 사용 (b 건너뜀)
```

---

## EXPLAIN 실행 계획

```sql
EXPLAIN SELECT * FROM users WHERE email = 'foo@example.com';
EXPLAIN ANALYZE SELECT ...;  -- 실제 실행 (PostgreSQL)
```

### MySQL EXPLAIN 주요 컬럼
| 컬럼 | 의미 |
|------|------|
| type | ALL(풀스캔) < index < range < ref < eq_ref < const |
| key | 사용된 인덱스 |
| rows | 예상 스캔 행 수 |
| Extra | Using filesort, Using temporary (나쁜 신호) |

`type: ALL` = 풀 테이블 스캔 → 인덱스 추가 필요

---

## 조인 (JOIN)

```sql
-- INNER JOIN: 양쪽 모두 존재하는 데이터
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: 왼쪽 테이블 기준 (오른쪽 없으면 NULL)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- 주문 없는 사용자 찾기
SELECT u.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

---

## 락 (Lock)

### 공유 락 vs 배타 락
- **공유 락 (Shared Lock, S)**: 읽기 전용. 여러 트랜잭션 동시 보유 가능
- **배타 락 (Exclusive Lock, X)**: 읽기/쓰기. 혼자만 보유 가능

```sql
-- 명시적 공유 락
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;  -- MySQL
SELECT * FROM users WHERE id = 1 FOR SHARE;           -- PostgreSQL

-- 명시적 배타 락
SELECT * FROM users WHERE id = 1 FOR UPDATE;
```

### 데드락
```sql
-- 트랜잭션 A: 1번 → 2번 순서로 락
-- 트랜잭션 B: 2번 → 1번 순서로 락
-- → 교착 상태!
```

---

## 주요 SQL 최적화

```sql
-- 1. SELECT * 대신 필요한 컬럼만
SELECT id, name FROM users WHERE ...;

-- 2. LIMIT 사용 (특히 개발/테스트)
SELECT * FROM large_table LIMIT 100;

-- 3. 서브쿼리 대신 JOIN
-- Bad
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE country = 'KR');
-- Good
SELECT o.* FROM orders o JOIN users u ON o.user_id = u.id WHERE u.country = 'KR';

-- 4. 페이지네이션 (OFFSET은 느림)
-- Bad (OFFSET이 크면 느림)
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 1000000;
-- Good (Cursor 기반)
SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT 20;
```

---

## 실무 포인트

- **슬로우 쿼리 로그**: MySQL `slow_query_log`, `long_query_time=1` 설정으로 느린 쿼리 탐지
- **Connection Pool**: 앱에서 DB 연결을 재사용. 연결 생성 비용 절감 (PgBouncer, HikariCP)
- **N+1 문제**: 루프 안에서 쿼리 반복 실행. JOIN 또는 IN으로 한 번에 조회
- **쓰기 지연 (Write-Behind)**: DB 직접 쓰기 대신 큐에 넣고 배치 처리 → 쓰기 처리량 향상
- **Read Replica**: 읽기 쿼리를 복제 서버로 분산. 마스터 부하 감소
- **VACUUM (PostgreSQL)**: 삭제/수정된 행의 공간 회수. 자동 VACUUM 튜닝 중요
