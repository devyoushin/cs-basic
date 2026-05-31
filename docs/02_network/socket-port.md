# 소켓 & 포트

## 소켓(Socket)이란?

소켓은 네트워크 통신의 **끝점(endpoint)**. 프로세스 간 통신을 위해 운영체제가 제공하는 추상화 인터페이스.

```
소켓 = IP 주소 + 포트 + 프로토콜
예: 192.168.1.100:8080 (TCP)
```

### 소켓의 식별

TCP 연결은 **4-tuple**로 유일하게 식별됨:
```
(클라이언트 IP, 클라이언트 포트, 서버 IP, 서버 포트)
예: (203.0.113.1, 54321, 10.0.0.1, 80)
```
→ 같은 서버:포트에도 클라이언트 IP/포트 조합이 다르면 별개의 연결

---

## 서버 소켓 생명주기

### BSD Socket API

```c
// 1. 소켓 생성
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
//                  ↑ IPv4    ↑ TCP

// 2. 주소 바인딩
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = INADDR_ANY;
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));

// 3. 연결 대기 (백로그 큐 설정)
listen(sockfd, 128);  // 128 = 백로그 크기

// 4. 연결 수락 (블로킹)
int connfd = accept(sockfd, NULL, NULL);
// → 새 소켓 fd 반환 (각 클라이언트마다 별도 fd)

// 5. 데이터 송수신
recv(connfd, buf, sizeof(buf), 0);
send(connfd, response, strlen(response), 0);

// 6. 연결 종료
close(connfd);
```

### 상태 흐름

```
socket()  → CLOSED
bind()    → 주소 할당
listen()  → LISTEN
accept()  → ESTABLISHED (블로킹 해제됨)
close()   → FIN_WAIT → TIME_WAIT → CLOSED
```

---

## 백로그 큐 (Backlog Queue)

`listen(sockfd, backlog)` 에서 backlog의 의미:

```
                    ┌─────────────────────┐
Client SYN ──────→  │  SYN Queue          │  (반완성 연결: SYN_RCVD)
                    │  (incompleted queue)│
                    └─────────┬───────────┘
                    3-way handshake 완료 ↓
                    ┌─────────────────────┐
accept() ←────────  │  Accept Queue       │  (완성된 연결: ESTABLISHED)
                    │  (completed queue)  │
                    └─────────────────────┘
```

- **Accept Queue 가득 참** → 새 SYN을 무시하거나 RST 전송 → 클라이언트 Connection Refused / Timeout
- 튜닝: `/proc/sys/net/core/somaxconn` (Accept Queue 최대값), `/proc/sys/net/ipv4/tcp_max_syn_backlog`

```bash
# 현재 백로그 설정 확인
cat /proc/sys/net/core/somaxconn     # 기본: 128
cat /proc/sys/net/ipv4/tcp_max_syn_backlog  # 기본: 512

# 고트래픽 서버 권장 설정 (sysctl.conf)
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
```

---

## 포트(Port)

### 포트 범위

| 범위 | 이름 | 설명 |
|------|------|------|
| 0–1023 | Well-known ports | 시스템 서비스 (root 필요) |
| 1024–49151 | Registered ports | 일반 애플리케이션 |
| 49152–65535 | Ephemeral ports | 클라이언트 임시 포트 |

### 임시 포트 (Ephemeral Port)
클라이언트가 서버에 연결할 때 OS가 자동 할당:
```bash
# 임시 포트 범위 확인/설정
cat /proc/sys/net/ipv4/ip_local_port_range
# 기본: 32768 60999

# 포트 고갈 방지를 위해 범위 확장
echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range
```

### 주요 포트 번호

| 포트 | 프로토콜 | 서비스 |
|------|----------|--------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP (대체) |
| 9092 | TCP | Kafka |
| 2379 | TCP | etcd (client) |

---

## TIME_WAIT & SO_REUSEADDR

### TIME_WAIT 문제
서버가 먼저 연결을 끊으면 TIME_WAIT 상태로 2MSL(~60초) 동안 포트 점유:

```bash
# TIME_WAIT 수 확인
ss -s | grep TIME-WAIT
netstat -an | grep TIME_WAIT | wc -l

# TIME_WAIT 포트 재사용 허용
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse   # outgoing 연결만 재사용
```

### SO_REUSEADDR
서버 재시작 시 "Address already in use" 오류 방지:
```c
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
// → 이전 TIME_WAIT 소켓이 있어도 bind() 성공
```

### SO_REUSEPORT (Linux 3.9+)
여러 프로세스/스레드가 같은 포트에 bind → 커널이 연결 분산:
```c
setsockopt(sockfd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
```
- Nginx (1.9.1+): `reuseport` 옵션으로 Worker별 Accept Queue 독립화 → 컨텍스트 스위칭 감소

---

## TCP 소켓 옵션 실무

### TCP_NODELAY (Nagle 알고리즘 비활성화)
Nagle: 작은 패킷을 모아서 전송 → 레이턴시 증가

```c
int flag = 1;
setsockopt(connfd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
```
- 인터랙티브 서비스 (실시간 게임, 금융) → TCP_NODELAY 권장
- 대용량 배치 전송 → Nagle 유지 (처리량 향상)

### SO_KEEPALIVE
유휴 연결의 생존 여부 확인:
```bash
# TCP KeepAlive 설정 (전역)
/proc/sys/net/ipv4/tcp_keepalive_time      # 유휴 후 첫 probe 시간 (기본: 7200초)
/proc/sys/net/ipv4/tcp_keepalive_intvl     # probe 간격 (기본: 75초)
/proc/sys/net/ipv4/tcp_keepalive_probes    # 실패 허용 횟수 (기본: 9)
```

### SO_LINGER
close() 시 송신 버퍼 처리 방식:
```c
struct linger lg = {1, 0};  // linger=on, timeout=0
setsockopt(connfd, SOL_SOCKET, SO_LINGER, &lg, sizeof(lg));
// → close() 즉시 RST 전송 (TIME_WAIT 없이 즉시 종료)
// 단, 데이터 유실 위험 — 신중하게 사용
```

---

## 소켓 버퍼 튜닝

```bash
# TCP 수신/송신 버퍼 크기
/proc/sys/net/ipv4/tcp_rmem    # min, default, max (수신)
/proc/sys/net/ipv4/tcp_wmem    # min, default, max (송신)
/proc/sys/net/core/rmem_max    # 수신 버퍼 최대값
/proc/sys/net/core/wmem_max    # 송신 버퍼 최대값

# 고대역폭 서버 권장 (sysctl.conf)
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
```

---

## 소켓 상태 진단

```bash
# 소켓 상태 전체 조회
ss -s                    # 요약 통계
ss -tulnp               # TCP/UDP listening 소켓
ss -tnp state ESTABLISHED  # 활성 연결만

# 특정 포트 연결 확인
ss -tnp sport = :443
ss -tnp dport = :3306

# 프로세스별 소켓
lsof -i TCP -n -P       # 모든 TCP 소켓 + PID
lsof -i :8080           # 특정 포트 사용 프로세스
lsof -p <PID> -i        # 특정 프로세스의 소켓

# 소켓 통계 (drops, errors)
ss -s
netstat -s | grep -i error

# 연결 상태별 카운트
ss -tn | awk '{print $1}' | sort | uniq -c
```

---

## 실무 포인트

- **CLOSE_WAIT 누적**: 서버가 FIN을 받았는데 close()를 안 함 → 애플리케이션 커넥션 풀 버그 의심. `ss -tn state CLOSE-WAIT`
- **TIME_WAIT 대량 발생**: 단기 HTTP 요청이 많은 경우 정상. 포트 고갈 위험 시 `tcp_tw_reuse`, `SO_REUSEADDR` 활용
- **Connection Refused**: 서버가 LISTEN 상태 아님 or Accept Queue 가득 참. `ss -tlnp`로 확인
- **포트 고갈**: `ip_local_port_range` 범위 내 포트가 모두 사용 중. `ss -s`로 TIME_WAIT 수 확인, 연결 풀 활용
- **커넥션 풀 (Connection Pool)**: HTTP Keep-Alive 또는 DB 커넥션 풀로 소켓 재사용 → 핸드셰이크 비용 절감
- **SYN Flood 방지**: `net.ipv4.tcp_syncookies = 1` 설정 (기본 활성화)
