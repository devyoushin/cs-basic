# CS Basic

CS 기초부터 Senior 레벨 심화 내용까지 정리한 저장소입니다.

---

## 목차

### 기초~중급

| # | 주제 | 파일 |
|---|------|------|
| 1 | Linux 기초 — 파일시스템, 핵심 명령어, systemd | [linux-fundamentals.md](01_os/linux-fundamentals.md) |
| 2 | 프로세스 & 스레드 — IPC, 동기화, 데드락 | [process-thread.md](01_os/process-thread.md) |
| 3 | 메모리 관리 — 가상 메모리, 페이징, OOM | [memory-management.md](01_os/memory-management.md) |
| 4 | OSI & TCP/IP — 3-way handshake, 소켓, 포트 | [osi-tcpip.md](02_network/osi-tcpip.md) |
| 5 | DNS — 조회 흐름, 레코드 타입, 진단 | [dns.md](02_network/dns.md) |
| 6 | HTTP/HTTPS — 버전 비교, 상태코드, TLS, CORS | [http-https.md](02_network/http-https.md) |
| 7 | 로드 밸런싱 — L4/L7, 알고리즘, 헬스체크 | [load-balancing.md](02_network/load-balancing.md) |
| 8 | SQL — ACID, 트랜잭션 격리, 인덱스, EXPLAIN | [sql-basics.md](03_database/sql-basics.md) |
| 9 | NoSQL & 캐싱 — Redis, 캐싱 전략, Elasticsearch | [nosql-caching.md](03_database/nosql-caching.md) |
| 10 | CAP 정리 & 분산 시스템 — BASE, 복제, Raft | [cap-theorem.md](04_distributed-systems/cap-theorem.md) |
| 11 | 메시지 큐 — Kafka, SQS, Pub/Sub 패턴 | [messaging.md](04_distributed-systems/messaging.md) |
| 12 | 확장성 — Scale Out, USE/RED 방법론, Auto Scaling | [scalability.md](05_system-design/scalability.md) |
| 13 | 신뢰성 — SLI/SLO/SLA, 에러 버짓, Circuit Breaker | [reliability.md](05_system-design/reliability.md) |
| 14 | 보안 — 시크릿 관리, IAM, TLS, 컨테이너 보안 | [security-basics.md](06_security/security-basics.md) |
| 15 | 모니터링 & 관찰가능성 — Metrics/Logs/Traces, Prometheus | [monitoring-logging.md](07_observability/monitoring-logging.md) |

### Deep Dive (Senior Level)

| 주제 | 파일 |
|------|------|
| Linux 커널 내부 — eBPF, syscall, NUMA, io_uring, OS 업그레이드, kdump, OOM, PSI, Seccomp, Network Namespace | [linux-kernel-internals.md](deepdive/linux-kernel-internals.md) |
| TCP 내부 — BBR/CUBIC, conntrack, XDP, 커널 튜닝 | [tcp-internals.md](deepdive/tcp-internals.md) |
| 데이터베이스 내부 — MVCC, WAL, B-Tree, 복제 | [database-internals.md](deepdive/database-internals.md) |
| 분산 시스템 심화 — Raft 내부, Consistent Hashing, HLC | [distributed-systems-advanced.md](deepdive/distributed-systems-advanced.md) |
| 컨테이너 내부 — Namespace, cgroup v2, Overlay FS, seccomp | [container-internals.md](deepdive/container-internals.md) |
| 성능 엔지니어링 — Flame Graph, perf, 레이턴시 분석 | [performance-engineering.md](deepdive/performance-engineering.md) |

---

## 구조

```
cs-basic/
├── 01_os/
├── 02_network/
├── 03_database/
├── 04_distributed-systems/
├── 05_system-design/
├── 06_security/
├── 07_observability/
└── deepdive/                  # Senior 레벨 심화
```

## 학습 순서 (추천)

```
Linux/OS → Network → Distributed Systems → Observability → Database → Security → System Design
                                                  ↓
                                          (기초 완료 후)
                                           Deep Dive
```

각 파일의 **실무 포인트** 섹션은 실제 운영 환경에서 바로 적용할 수 있는 내용입니다.
