# 파일 디스크립터 & I/O 모델

## 파일 디스크립터 (File Descriptor)

Linux에서 **모든 것은 파일**. 파일, 소켓, 파이프, 디바이스 모두 FD로 접근.

```
프로세스 파일 디스크립터 테이블
┌────┬────────────────────────────┐
│ FD │ 대상                        │
├────┼────────────────────────────┤
│  0 │ stdin  (표준 입력)          │
│  1 │ stdout (표준 출력)          │
│  2 │ stderr (표준 에러)          │
│  3 │ /var/log/app.log (파일)     │
│  4 │ TCP socket :8080 (소켓)     │
│  5 │ pipe (파이프)               │
└────┴────────────────────────────┘
```

### FD 관련 커널 자료구조
```
프로세스 → File Descriptor Table (프로세스별)
              ↓
           Open File Table (시스템 전체)
              ↓
           Inode Table (파일시스템)
```
- **FD**: 프로세스가 사용하는 정수 인덱스 (0, 1, 2, ...)
- **Open File Entry**: 오프셋, 플래그, 파일 상태 등 보관
- **fork() 후**: 부모/자식이 같은 Open File Entry를 공유 (오프셋 공유)

---

## FD 한도

```bash
# 프로세스별 FD 한도
ulimit -n                          # 현재 세션 소프트 한도 (기본: 1024)
ulimit -Hn                         # 하드 한도

# 특정 프로세스의 FD 한도 및 사용량
cat /proc/<PID>/limits | grep "open files"
ls /proc/<PID>/fd | wc -l          # 현재 열린 FD 수

# 시스템 전체 FD 한도
cat /proc/sys/fs/file-max          # 시스템 전체 최대 FD 수
cat /proc/sys/fs/file-nr           # 현재 사용 중 / 최대
```

### 한도 높이기
```bash
# 세션 임시 변경
ulimit -n 65536

# 영구 변경 (/etc/security/limits.conf)
*    soft    nofile    65536
*    hard    nofile    65536

# systemd 서비스 단위 설정 (.service 파일)
[Service]
LimitNOFILE=65536
```

### FD 고갈 증상
- `Too many open files` 에러
- 새 소켓/파일 생성 실패 → 503, Connection Refused
- Nginx: `worker_rlimit_nofile` 설정 필요

---

## I/O 모델 비교

### 1. Blocking I/O

```
Application         Kernel
    │                  │
    │── recvfrom() ──→ │
    │   (BLOCK)        │  데이터 도착 대기
    │                  │  ↓ 데이터 복사
    │ ←── 응답 ──────  │
    │  (계속 진행)      │
```
- 스레드가 I/O 완료 전까지 **대기** (CPU를 양보)
- 구현 단순, 동시 요청 증가 시 스레드 수 비례 증가
- **Apache Prefork MPM**: 요청당 프로세스 → 대규모에 비효율

---

### 2. Non-Blocking I/O

```
Application         Kernel
    │                  │
    │── recvfrom() ──→ │  (데이터 없음)
    │ ←── EAGAIN ────  │  즉시 반환
    │  (다른 작업)      │
    │── recvfrom() ──→ │  (폴링)
    │ ←── EAGAIN ────  │
    │── recvfrom() ──→ │  데이터 있음
    │ ←── 응답 ──────  │
```
- I/O 준비 안 됐으면 즉시 `EAGAIN` 반환
- **Busy-waiting** (CPU 낭비) → 단독으로 잘 안 씀
- I/O 멀티플렉싱과 조합해서 사용

```c
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

---

### 3. I/O Multiplexing (select/poll/epoll)

**select** (POSIX 표준, FD 1024 개 제한):
```c
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);
select(maxfd+1, &readfds, NULL, NULL, &timeout);
// → 준비된 FD가 있으면 반환
// 단점: 매번 전체 FD 집합을 커널에 전달, O(n) 탐색
```

**epoll** (Linux 전용, 고성능):
```c
// epoll 인스턴스 생성
int epfd = epoll_create1(0);

// FD 등록
struct epoll_event ev;
ev.events = EPOLLIN;     // 읽기 이벤트 감시
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// 이벤트 대기
struct epoll_event events[64];
int n = epoll_wait(epfd, events, 64, -1);
// → 준비된 이벤트만 반환 (O(1))
for (int i = 0; i < n; i++) {
    handle(events[i].data.fd);
}
```

| | select | poll | epoll |
|--|--------|------|-------|
| FD 제한 | 1024 | 무제한 | 무제한 |
| 시간 복잡도 | O(n) | O(n) | O(1) |
| 이벤트 전달 | 전체 FD 스캔 | 전체 FD 스캔 | 변경된 FD만 |
| 커널 지원 | POSIX | POSIX | Linux 전용 |
| 사용처 | 레거시 | 일부 구형 | Nginx, Node.js, Redis |

---

### 4. epoll 동작 방식 심화

**Level Triggered (LT, 기본)** vs **Edge Triggered (ET)**:
```
데이터: [A A A]  →  [A A A A A]  →  []

LT: 버퍼에 데이터 있는 동안 계속 이벤트 발생 (안전, 단순)
ET: 상태 변화 시에만 이벤트 발생 (효율적, 구현 복잡)
    → ET는 반드시 Non-Blocking + 루프로 전체 읽기 필요
```

**EPOLLET** (Edge Triggered) + Non-Blocking 조합이 Nginx의 핵심.

---

### 5. 비동기 I/O (AIO / io_uring)

**POSIX AIO**:
```
Application         Kernel
    │                  │
    │── aio_read() ──→ │  (즉시 반환)
    │  (다른 작업)      │  백그라운드 I/O 진행
    │                  │  ↓ 완료 시 시그널/콜백
    │ ←── 완료 알림 ─  │
```

**io_uring** (Linux 5.1+, 최신 고성능):
- 시스템 콜 없이 공유 링 버퍼로 커널과 통신
- 배치 처리, 제로카피 지원
- 데이터베이스(ScyllaDB, RocksDB), 고성능 서버에서 채택 중

---

## I/O 모델 비교 요약

| 모델 | 블로킹 | 특징 | 사용 사례 |
|------|--------|------|---------|
| Blocking I/O | O | 단순, 스레드당 연결 | Apache Prefork |
| Non-Blocking I/O | X | Busy-waiting | 단독 사용 희귀 |
| select/poll | 대기 중만 | FD 제한/선형 탐색 | 레거시 코드 |
| epoll | 대기 중만 | O(1), 고성능 | Nginx, Node.js, Redis |
| AIO/io_uring | X | 완전 비동기 | DB, 고성능 스토리지 |

---

## 서버 아키텍처와 I/O 모델

### Apache Prefork MPM (Blocking I/O)
```
Master Process
  ├── Worker 1 (1개 연결 처리, Blocking)
  ├── Worker 2
  └── Worker N
```
- 요청당 프로세스/스레드 → 동시 10K 연결 = 10K 스레드 → 메모리 고갈 (C10K 문제)

### Nginx Event-Driven (epoll)
```
Master Process
  └── Worker 1 (epoll로 수천 연결 동시 처리)
  └── Worker 2 (CPU 코어 수만큼)
```
- 1 Worker가 epoll로 수천 연결 관리 → C10K 해결
- Non-Blocking + Edge Triggered + Worker 수 = CPU 코어 수

### Node.js (epoll + libuv)
```
Event Loop (Single Thread)
  │── epoll로 I/O 감시
  │── I/O 완료 시 콜백 실행
  └── libuv 스레드풀 (파일 I/O, DNS 등 4개 기본)
```

---

## 실무 포인트

- **ulimit -n**: 고트래픽 서버의 첫 번째 튜닝 포인트. Nginx `worker_rlimit_nofile`, Java `-Djava.net.preferIPv4Stack`
- **epoll LT vs ET**: Nginx는 ET 모드 + Non-Blocking. ET는 미처리 데이터 유실 위험 → 루프로 완전 소비 필수
- **`/proc/sys/fs/file-nr`** 모니터링: 시스템 전체 FD 사용량. 한도의 80% 이상이면 증설 필요
- **FD 누수 탐지**: `lsof -p <PID> | wc -l` 증가 추세 확인. Java는 `lsof -p <PID> | grep -v REG` (소켓/파이프 위주)
- **C10K 문제**: 동시 10,000 연결을 처리하기 위한 아키텍처 설계. epoll 기반 이벤트 모델로 해결
- **Tomcat**: NIO Connector (epoll 기반), BIO(레거시) → NIO/NIO2 권장
