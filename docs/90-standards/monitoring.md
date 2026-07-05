# 모니터링 및 확인 기준

CS 개념을 실무에서 확인하는 명령어 및 도구 작성 기준입니다.

---

## 1. OS/커널 확인 도구

```bash
# 프로세스 및 시스템 상태
top -b -n1                          # 스냅샷
vmstat 1 5                          # 메모리/CPU 5초간
iostat -xz 1                        # 디스크 I/O

# 메모리 상세
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|Cached"

# 파일 디스크립터
lsof -p <PID> | wc -l               # 프로세스 FD 수
```

## 2. 네트워크 확인 도구

```bash
# TCP 연결 상태
ss -tlnp                            # 리스닝 포트
ss -s                               # 연결 통계

# DNS 확인
dig +trace example.com              # DNS 조회 경로
dig @8.8.8.8 example.com           # 특정 resolver 사용
```

## 3. JVM 모니터링

```bash
# GC 로그 활성화
# -Xlog:gc*:file=gc.log:time,level,tags

# 힙 덤프
jmap -heap <PID>                    # 힙 요약
jstack <PID>                        # 스레드 상태
```

## 4. 데이터베이스 모니터링

```sql
-- MySQL 슬로우 쿼리 확인
SHOW VARIABLES LIKE 'slow_query_log%';
SELECT * FROM information_schema.PROCESSLIST WHERE TIME > 5;

-- 인덱스 사용률
EXPLAIN FORMAT=JSON SELECT ...;
```

## 5. 문서별 필수 포함 항목

| CS 분야 | 필수 확인 명령어 |
|---------|--------------|
| OS | `top`, `vmstat`, `dmesg` |
| Network | `ss`, `netstat -s`, `tcpdump` |
| Database | `EXPLAIN`, `SHOW PROCESSLIST` |
| JVM | `jstack`, `jmap`, GC 로그 |
| Container | `kubectl top`, cgroup 통계 |
