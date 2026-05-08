# CS Basic

SRE/DevOps 실무와 연결되는 CS 기초~Senior 레벨 심화 내용을 한국어로 정리한 저장소입니다.

---

## 디렉토리 구조

```
cs-basic/
├── 01_os/                     # 운영체제
├── 02_network/                # 네트워킹
├── 03_database/               # 데이터베이스
├── 04_distributed-systems/    # 분산 시스템
├── 05_system-design/          # 시스템 설계
├── 06_security/               # 보안
├── 07_observability/          # 모니터링 & 관찰가능성
├── 08_jvm/                    # JVM 기초
├── 09_container/              # 컨테이너 & 쿠버네티스
└── deepdive/                  # Senior 레벨 심화
```

---

## 학습 순서 (추천)

```
OS → Network → Distributed Systems → Observability → Database → JVM → Security → System Design
                                            ↓
                                      (기초 완료 후)
                                        Deep Dive
```

**우선순위 이유**
1. **OS** — 모든 인프라의 기반
2. **Network** — 트래픽 흐름과 장애 원인 파악에 필수
3. **Distributed Systems** — SRE의 핵심 도메인
4. **Observability** — 장애 감지 및 대응
5. **Database** — 성능 병목 원인 분석
6. **JVM** — Java/Tomcat WAS 운영 필수
7. **Security** — 인프라 보안 hardening
8. **System Design** — 아키텍처 설계 및 리뷰

---

## 문서 목록

### 01_os — 운영체제

| 파일 | 핵심 주제 |
|------|----------|
| [linux-fundamentals.md](01_os/linux-fundamentals.md) | 파일시스템, 핵심 명령어, systemd, strace/lsof/iostat |
| [process-thread.md](01_os/process-thread.md) | PCB, 스케줄링 알고리즘, IPC, 동기화, Thread Dump |
| [memory-management.md](01_os/memory-management.md) | 가상 메모리, 페이지 교체 알고리즘, OOM, Thrashing |
| [fd-io-models.md](01_os/fd-io-models.md) | FD, Blocking/Non-Blocking/Async I/O, epoll, C10K |
| [syscall-kernel.md](01_os/syscall-kernel.md) | 시스템 콜, 커널/유저 모드 전환, strace |

### 02_network — 네트워킹

| 파일 | 핵심 주제 |
|------|----------|
| [osi-tcpip.md](02_network/osi-tcpip.md) | OSI/TCP 모델, 3-way handshake, 혼잡제어 (CWND/Slow Start) |
| [dns.md](02_network/dns.md) | DNS 쿼리 흐름, TTL, 레코드 타입, 진단 도구 |
| [http-https.md](02_network/http-https.md) | HTTP/1.1~3, QUIC, TLS 핸드셰이크, SNI, CORS |
| [load-balancing.md](02_network/load-balancing.md) | L4/L7, 알고리즘 (RR/LC/IP Hash), 헬스체크 |
| [socket-port.md](02_network/socket-port.md) | 소켓 API, 백로그 큐, 소켓 옵션 튜닝 |
| [iptables-nat.md](02_network/iptables-nat.md) | Netfilter, iptables 체인, NAT, conntrack |
| [cdn.md](02_network/cdn.md) | CDN 동작 원리, Edge 캐시, 캐싱 전략 |
| [vpn.md](02_network/vpn.md) | VPN 터널링, IPsec, WireGuard |
| [advanced-networking.md](02_network/advanced-networking.md) | BGP, OSPF, ECMP, SR-IOV, DPDK (심화) |

### 03_database — 데이터베이스

| 파일 | 핵심 주제 |
|------|----------|
| [sql-basics.md](03_database/sql-basics.md) | ACID, 트랜잭션 격리 수준, 인덱스, EXPLAIN |
| [transaction-lock.md](03_database/transaction-lock.md) | WAL, 락 종류, 갭락, 데드락, MVCC |
| [nosql-caching.md](03_database/nosql-caching.md) | Redis, MongoDB, 캐싱 패턴 (Cache-Aside/Write-Through) |

### 04_distributed-systems — 분산 시스템

| 파일 | 핵심 주제 |
|------|----------|
| [cap-theorem.md](04_distributed-systems/cap-theorem.md) | CAP 정리, BASE vs ACID, 일관성 모델, 복제 |
| [messaging.md](04_distributed-systems/messaging.md) | Kafka, 메시지 큐, Pub/Sub, 이벤트 스트리밍 |

### 05_system-design — 시스템 설계

| 파일 | 핵심 주제 |
|------|----------|
| [scalability.md](05_system-design/scalability.md) | 수평/수직 확장, 샤딩, 파티셔닝, Auto Scaling |
| [reliability.md](05_system-design/reliability.md) | SLI/SLO/SLA, 에러 버짓, Circuit Breaker, 장애 설계 |
| [cicd-deployment.md](05_system-design/cicd-deployment.md) | CI/CD 파이프라인, Blue-Green/Canary/Rolling 배포 |
| [backup.md](05_system-design/backup.md) | 파일 백업 vs 스냅샷 백업, RTO/RPO, 백업 전략 |

### 06_security — 보안

| 파일 | 핵심 주제 |
|------|----------|
| [security-basics.md](06_security/security-basics.md) | 위협 모델링, 시크릿 관리, IAM, TLS, 컨테이너 보안 |
| [auth.md](06_security/auth.md) | JWT 심화, OAuth 2.0, PKCE, OIDC, RBAC |

### 07_observability — 모니터링 & 관찰가능성

| 파일 | 핵심 주제 |
|------|----------|
| [monitoring-logging.md](07_observability/monitoring-logging.md) | Metrics/Logs/Traces, Prometheus, OpenTelemetry |

### 08_jvm — JVM

| 파일 | 핵심 주제 |
|------|----------|
| [jvm-architecture.md](08_jvm/jvm-architecture.md) | 클래스 로더, 실행 엔진, JIT 컴파일러 |
| [jvm-memory.md](08_jvm/jvm-memory.md) | Heap/Stack/Metaspace/Native Memory |
| [gc-basics.md](08_jvm/gc-basics.md) | GC 알고리즘 (G1/ZGC/Shenandoah), 튜닝 기초 |

### 09_container — 컨테이너 & 쿠버네티스

| 파일 | 핵심 주제 |
|------|----------|
| [docker-basics.md](09_container/docker-basics.md) | Namespace/cgroup/OverlayFS, 이미지 레이어, 네트워킹 |
| [kubernetes-basics.md](09_container/kubernetes-basics.md) | 아키텍처, 스케줄링, Service/Ingress, RBAC, NetworkPolicy |

### deepdive — Senior 레벨 심화

| 파일 | 핵심 주제 |
|------|----------|
| [linux-kernel-internals.md](deepdive/linux-kernel-internals.md) | eBPF, io_uring, NUMA, kdump, OOM, PSI, Seccomp, Network Namespace |
| [tcp-internals.md](deepdive/tcp-internals.md) | BBR/CUBIC 혼잡제어, conntrack 내부, XDP, 커널 파라미터 튜닝 |
| [tcp-deep-dive.md](deepdive/tcp-deep-dive.md) | TCP 상태머신, 3/4-way handshake 커널 내부, 흐름제어/혼잡제어 심화, VPC Flow Log 분석, sysctl 완전 정리 |
| [database-internals.md](deepdive/database-internals.md) | MVCC 내부, WAL 구조, B-Tree 분할, 복제 내부 |
| [distributed-systems-advanced.md](deepdive/distributed-systems-advanced.md) | Raft 내부 구현, Consistent Hashing, HLC |
| [container-internals.md](deepdive/container-internals.md) | Namespace 종류, cgroup v2, OverlayFS, seccomp BPF |
| [performance-engineering.md](deepdive/performance-engineering.md) | Flame Graph, perf/bpftrace, 레이턴시 분포 분석 |
| [kubernetes-deep-dive.md](deepdive/kubernetes-deep-dive.md) | etcd Raft, kube-proxy iptables/IPVS, 스케줄러 프레임워크, CNI/CRI 내부, cgroup QoS |
| [linux-memory-internals.md](deepdive/linux-memory-internals.md) | 페이지 테이블, COW, OOM Killer, NUMA, HugePage/THP, cgroup 메모리 컨트롤러 |
| [observability-deep-dive.md](deepdive/observability-deep-dive.md) | eBPF 트레이싱, Prometheus TSDB/PromQL, OpenTelemetry 내부, Tail Sampling, SLO 번 레이트 |
| [storage-io-internals.md](deepdive/storage-io-internals.md) | 블록 I/O 스택, io_uring, NVMe 큐, 파일시스템 저널링, EBS/S3 내부 동작 |

---

## 사용 방법

- 각 파일은 **개념 설명 + 실무 적용 포인트**로 구성됩니다.
- `# 실무 포인트` / `# 실무 심화 포인트` 섹션은 면접 및 실제 운영에서 바로 쓸 수 있는 내용입니다.
- Claude에게 특정 주제 심화 내용 추가를 요청할 수 있습니다.
