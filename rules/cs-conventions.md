# CS 코드 작성 규칙

이 저장소의 코드 예시 작성 규칙입니다. CS 개념을 설명하기 위한 코드이므로 단순하고 명확하게 작성합니다.

---

## 1. 언어별 코드 규칙

### Java (JVM, 동시성 관련)
```java
// JVM 힙 옵션 예시
// -Xms512m      초기 힙 크기
// -Xmx2048m     최대 힙 크기
// -XX:+UseG1GC  G1 GC 사용

// 동시성: synchronized vs ReentrantLock
public class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();        // 락 획득
        try {
            count++;
        } finally {
            lock.unlock();  // 반드시 finally에서 해제
        }
    }
}
```

### SQL (데이터베이스 관련)
```sql
-- EXPLAIN으로 실행 계획 확인
EXPLAIN SELECT * FROM orders
WHERE user_id = 1
  AND created_at > '2024-01-01';

-- 복합 인덱스 생성
CREATE INDEX idx_orders_user_created
ON orders(user_id, created_at);
```

### Bash (Linux, 시스템 확인)
```bash
# TCP 연결 상태 확인
ss -tlnp | grep :8080

# 프로세스 메모리 확인
cat /proc/<PID>/status | grep -E "VmRSS|VmSize"
```

## 2. 다이어그램 표현

ASCII 아트로 간단한 구조 표현:
```
Client → [Load Balancer] → [Web Server] → [DB]
                                       ↗
                              [Cache]
```

## 3. 플레이스홀더 표기법

| 타입 | 형식 |
|------|------|
| 포트 | `<PORT>` |
| IP | `<IP_ADDR>` |
| PID | `<PID>` |
| DB명 | `<DATABASE>` |

## 4. 성능 수치 표기

- 수치는 반드시 단위 포함: `100ms`, `1GB`, `10,000 QPS`
- 범위 표기: `100ms ~ 200ms`
- 출처 명시: (측정 환경: AWS t3.medium)
