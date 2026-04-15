# 트랜잭션 & 락 심화

## 트랜잭션의 물리적 구현: WAL (Write-Ahead Logging)

ACID의 Atomicity와 Durability는 WAL로 구현됨.

```
트랜잭션 COMMIT 흐름:
① BEGIN
② 변경사항 → 메모리 내 버퍼 페이지 수정
③ 변경 내용 → WAL(Redo Log)에 순차 기록 (fsync)
④ COMMIT 완료 응답
⑤ 이후 백그라운드로 버퍼 → 디스크 플러시 (Checkpoint)

Recovery 시:
① WAL 재실행 (Redo): 커밋된 트랜잭션 복원
② Undo Log 실행: 미완료 트랜잭션 롤백
```

**WAL이 순차 쓰기인 이유**: 랜덤 I/O보다 순차 I/O가 디스크에서 훨씬 빠름.
버퍼에 쓰고 나중에 플러시 → 빠른 응답 + WAL로 내구성 보장.

---

## Lock의 종류와 호환성

### 기본 락 유형

| 락 | 기호 | 허용 동시 작업 | 목적 |
|----|------|-------------|------|
| **S (Shared)** | 공유 락 | 다른 S락은 허용 | 읽기 |
| **X (Exclusive)** | 배타 락 | 아무것도 허용 안 함 | 쓰기 |
| **IS (Intention Shared)** | 의도 공유 | IS, S, IX 허용 | 하위에 S락 예고 |
| **IX (Intention Exclusive)** | 의도 배타 | IS, IX만 허용 | 하위에 X락 예고 |
| **SIX** | S + IX | IS만 허용 | 전체 읽기 + 일부 쓰기 |

### 락 호환성 행렬

```
         IS   IX    S   SIX   X
IS        O    O    O    O    X
IX        O    O    X    X    X
S         O    X    O    X    X
SIX       O    X    X    X    X
X         X    X    X    X    X
```

**의도 락(IS/IX)의 역할**: 계층적 락에서 상위 노드에 미리 표시.
```
테이블에 IX 락 → "이 테이블의 특정 행에 X락을 걸 예정"
→ 다른 트랜잭션이 테이블 전체 S락을 걸려 할 때 즉시 충돌 감지
→ 행 수준까지 내려가지 않아도 됨 (성능)
```

---

## InnoDB 락 구조 심화

### 레코드 락 (Record Lock)

인덱스 레코드에 걸리는 락. **인덱스가 없으면 전체 테이블 락**.

```sql
-- id에 인덱스 있음: id=5 레코드에만 락
SELECT * FROM orders WHERE id = 5 FOR UPDATE;

-- email에 인덱스 없음: 풀 테이블 스캔 → 모든 레코드에 락
SELECT * FROM orders WHERE email = 'a@b.com' FOR UPDATE;
-- → 심각한 동시성 문제, 인덱스 필수
```

### 갭 락 (Gap Lock)

인덱스 레코드 **사이의 공간**에 걸리는 락. Phantom Read 방지.

```
인덱스 값: [1, 5, 10, 15, 20]

SELECT * FROM t WHERE id BETWEEN 5 AND 15 FOR UPDATE;
→ 레코드 락: 5, 10, 15
→ 갭 락:    (5, 10), (10, 15)  ← 이 범위에 INSERT 차단

다른 트랜잭션:
INSERT INTO t VALUES (7);   -- 갭 (5,10) 내 → 대기
INSERT INTO t VALUES (25);  -- 갭 범위 밖 → 즉시 성공
```

**갭 락은 쓰기 쿼리 간에도 충돌하지 않음**:
```
트랜잭션 A: 갭 (5,10) 갭 락
트랜잭션 B: 같은 갭 (5,10) 갭 락 → 공유 가능 (갭 락끼리는 호환)
트랜잭션 C: 갭 (5,10)에 INSERT → 차단 (INSERT는 X락 필요)
```

### 넥스트 키 락 (Next-Key Lock)

레코드 락 + 갭 락의 조합. InnoDB REPEATABLE READ의 기본 락.

```
인덱스: [1, 5, 10, ∞(supremum)]
넥스트 키 락 범위:
  (-∞, 1], (1, 5], (5, 10], (10, ∞)

SELECT * FROM t WHERE id = 5 FOR UPDATE;
→ 넥스트 키 락: (1, 5] = 갭(1,5) + 레코드(5)
```

### INSERT 의도 락 (Insert Intention Lock)

INSERT 전에 갭 락과의 호환성 체크용. INSERT끼리는 같은 갭이라도 서로 다른 위치면 충돌 안 함.

```
갭 (5, 10):
  INSERT 7: Insert Intention Lock on (5,10)
  INSERT 8: Insert Intention Lock on (5,10) → 7과 8은 위치 달라 공존 가능

  하지만 갭 락이 (5,10)에 있으면 → INSERT 7, 8 모두 차단
```

---

## 2단계 락킹 (2PL: Two-Phase Locking)

직렬화 가능한(Serializable) 스케줄을 보장하는 프로토콜.

```
성장 단계 (Growing Phase): 락 획득만 가능, 해제 불가
수축 단계 (Shrinking Phase): 락 해제만 가능, 획득 불가

트랜잭션: [락 획득, 획득, 획득] | [락 해제, 해제, 해제]
                              ↑
                          Lock Point
```

InnoDB는 **Strict 2PL**: 모든 락을 트랜잭션 종료(COMMIT/ROLLBACK) 시에만 해제.
→ 가장 강한 격리, 데드락 가능성 높음.

---

## 데드락 (Deadlock)

### 발생 조건과 예시

```
트랜잭션 A:           트랜잭션 B:
LOCK ROW 1 (X)       LOCK ROW 2 (X)
↓                    ↓
WAIT FOR ROW 2 ←→   WAIT FOR ROW 1
(순환 대기 → Deadlock)
```

**실제 시나리오**:
```sql
-- 트랜잭션 A (주문 처리)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- ROW 1 X락
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- ROW 2 대기

-- 트랜잭션 B (환불 처리, 동시에)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- ROW 2 X락
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- ROW 1 대기
-- → DEADLOCK
```

### InnoDB 데드락 감지와 처리

InnoDB는 **Wait-For Graph**를 유지하며 사이클(순환 대기) 자동 감지.

```
Wait-For Graph:
A → B (A가 B가 가진 락을 대기)
B → A (B가 A가 가진 락을 대기)
→ 사이클 감지 → 둘 중 비용이 낮은 트랜잭션 롤백 (undone 레코드 수 기준)
```

```sql
-- 데드락 정보 확인
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션 확인

-- 데드락 로깅 활성화
SET GLOBAL innodb_print_all_deadlocks = ON;
-- → error log에 모든 데드락 기록
```

### 데드락 방지 패턴

**1. 일관된 락 순서 (가장 중요)**
```sql
-- Bad: 서로 다른 순서로 락
-- A: 계좌1 → 계좌2
-- B: 계좌2 → 계좌1

-- Good: 항상 낮은 ID 순서로
-- A와 B 모두: min(id1,id2) 먼저 락, max(id1,id2) 나중에 락
UPDATE accounts SET balance = balance - 100 WHERE id = MIN(1,2);
UPDATE accounts SET balance = balance + 100 WHERE id = MAX(1,2);
```

**2. 트랜잭션을 짧게**
```sql
-- Bad: 트랜잭션 내에서 긴 처리
BEGIN;
UPDATE orders SET status = 'processing' WHERE id = 1;
[외부 API 호출 - 수 초 대기]  ← 락 유지 중
UPDATE inventory SET quantity = quantity - 1;
COMMIT;

-- Good: 외부 호출 전후로 트랜잭션 분리
result = external_api_call();  -- 락 없이
BEGIN;
UPDATE orders SET status = 'processing' WHERE id = 1;
UPDATE inventory SET quantity = quantity - 1;
COMMIT;
```

**3. SELECT ... FOR UPDATE 대신 낙관적 락**
```sql
-- 낙관적 락: 버전 번호로 충돌 감지
SELECT id, quantity, version FROM inventory WHERE id = 1;
-- (version = 5 확인)

UPDATE inventory
SET quantity = quantity - 1, version = version + 1
WHERE id = 1 AND version = 5;  -- version이 바뀌었으면 0 rows affected

-- 0 rows → 다른 트랜잭션이 먼저 수정 → 재시도
-- 충돌이 드물 때 유리 (락 경합 없음)
```

---

## 락 대기 & 타임아웃

```sql
-- 락 대기 타임아웃 설정 (기본 50초)
SET SESSION innodb_lock_wait_timeout = 10;  -- 10초

-- 현재 락 대기 상황 조회
SELECT * FROM information_schema.INNODB_TRX;         -- 활성 트랜잭션
SELECT * FROM information_schema.INNODB_LOCK_WAITS;  -- 대기 상황
SELECT * FROM information_schema.INNODB_LOCKS;       -- 보유 락

-- MySQL 8.0+
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

---

## MVCC와 락의 관계

MVCC(Multi-Version Concurrency Control)는 읽기 작업에 락이 필요 없게 함.

```
시간축:
T=1: INSERT INTO t VALUES (1, 'a')   → 버전 1
T=2: UPDATE t SET name='b' WHERE id=1 → 버전 2 생성, 버전 1은 언두 로그에
T=3: SELECT * FROM t WHERE id=1 (트랜잭션 시작 T=1.5 시점):
     → 버전 2가 있지만 T=1.5 시점엔 버전 1이 최신
     → 언두 로그에서 버전 1 읽음
     → 락 없이 일관된 읽기 (Consistent Read)
```

**MVCC가 없으면**: 읽기도 S락 필요 → 쓰기와 충돌 → 동시성 급감.

```sql
-- MVCC 읽기 (락 없음): Consistent Read
SELECT * FROM t WHERE id = 1;

-- 현재 버전 읽기 + 락: Locking Read
SELECT * FROM t WHERE id = 1 FOR UPDATE;   -- X락
SELECT * FROM t WHERE id = 1 FOR SHARE;    -- S락 (MySQL 8.0+)
SELECT * FROM t WHERE id = 1 LOCK IN SHARE MODE;  -- S락 (구버전)
```

---

## 격리 수준별 락 동작 (InnoDB)

| 격리 수준 | 넥스트 키 락 | 갭 락 | Phantom Read |
|---------|-----------|------|------------|
| READ UNCOMMITTED | X | X | 발생 |
| READ COMMITTED | X (레코드 락만) | X | 발생 |
| REPEATABLE READ | O | O | InnoDB는 방지 |
| SERIALIZABLE | O | O | 방지, 모든 SELECT에 S락 |

**READ COMMITTED에서 갭 락 없는 이유**:
- 갭 락 없음 → INSERT 허용 → Phantom Read 발생 가능
- 하지만 동시성 높음, 데드락 가능성 낮음
- PostgreSQL 기본값 (다른 MVCC 방식으로 Phantom Read 방지)

---

## 실무 포인트

- **느린 쿼리 = 긴 락 보유**: `EXPLAIN`으로 풀 테이블 스캔 확인. 인덱스 없는 컬럼으로 `FOR UPDATE`는 금물
- **트랜잭션 내 사용자 인터랙션 금지**: 웹 요청 처리 중 트랜잭션 열어두고 응답 보낸 후 commit → 락이 수 초 이상 유지 → 데드락 발생
- **`SHOW PROCESSLIST`**: `Lock wait` 상태 쿼리가 많으면 락 경합 심각. `KILL <id>`로 긴 트랜잭션 제거
- **갭 락과 AUTO_INCREMENT**: `innodb_autoinc_lock_mode=2` (MySQL 8 기본) → 가장 빠른 AUTO_INCREMENT. 그러나 binlog `STATEMENT` 형식과 호환 불가 → `ROW` 형식 사용 권장
- **데드락 발생 시 대응**: 애플리케이션 레벨에서 `deadlock detected` 에러 잡아서 재시도 (1205 오류). 재시도 없으면 요청 유실
- **PostgreSQL vs MySQL 락**: PostgreSQL은 MVCC + S2PL, 넥스트 키 락 없음. `SELECT FOR UPDATE` + `SKIP LOCKED`으로 비관적 큐 구현 가능
