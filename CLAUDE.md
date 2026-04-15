# CS Basic

## 목적
CS 기초 지식을 체계적으로 정리한 저장소입니다.

## 디렉토리 구조

```
cs-basic/
├── CLAUDE.md                  # 이 파일 (프로젝트 가이드)
├── 01_os/                     # 운영체제
│   ├── linux-fundamentals.md  # 명령어, 진단 도구 (strace/lsof/iostat)
│   ├── process-thread.md      # PCB, 스케줄링 알고리즘, 동기화, Thread Dump
│   ├── memory-management.md   # 가상 메모리, 페이지 교체 알고리즘, Thrashing
│   ├── fd-io-models.md        # FD, Blocking/Non-Blocking/epoll, C10K
│   └── syscall-kernel.md      # 시스템 콜, 커널/유저 모드, strace
├── 02_network/                # 네트워킹
│   ├── osi-tcpip.md           # OSI/TCP, TCP 혼잡제어 (CWND/Slow Start)
│   ├── dns.md
│   ├── http-https.md          # HTTP/1.1~3, QUIC, TLS, SNI
│   ├── load-balancing.md      # LB 알고리즘 상세 (RR, LC, IP Hash 등)
│   ├── socket-port.md         # 소켓 API, 백로그, 소켓 옵션 튜닝
│   ├── iptables-nat.md        # Netfilter, iptables, NAT, conntrack
│   ├── cdn.md
│   └── vpn.md
├── 03_database/               # 데이터베이스
│   ├── sql-basics.md          # ACID, 격리 수준
│   ├── transaction-lock.md    # WAL, 락 종류, 갭락, 데드락, MVCC
│   ├── indexing-optimization.md
│   └── nosql-caching.md
├── 04_distributed-systems/    # 분산 시스템
│   ├── cap-theorem.md
│   ├── consensus.md
│   └── messaging.md
├── 05_system-design/          # 시스템 설계
│   ├── scalability.md
│   ├── reliability.md
│   └── cicd-deployment.md     # CI/CD, 배포 전략 (Blue-Green/Canary/Rolling)
├── 06_security/               # 보안
│   ├── security-basics.md
│   └── auth.md                # JWT 심화, OAuth 2.0, PKCE, RBAC
├── 07_observability/          # 모니터링 & 관찰가능성
│   └── monitoring-logging.md
├── 08_jvm/                    # JVM 기초 (Java/WAS 운영)
│   ├── jvm-architecture.md    # 클래스 로더, 실행 엔진, JIT 컴파일러
│   ├── jvm-memory.md          # Heap/Stack/Metaspace/Native Memory
│   └── gc-basics.md           # GC 알고리즘, GC 종류, 튜닝 기초
├── 09_container/              # 컨테이너 & 쿠버네티스
│   ├── docker-basics.md       # Namespace/cgroup/OverlayFS, 네트워킹
│   └── kubernetes-basics.md   # 아키텍처, 스케줄링, Service/RBAC/NetworkPolicy
└── deepdive/                  # Senior 레벨 심화
    ├── linux-kernel-internals.md    # eBPF, syscall, NUMA, io_uring
    ├── tcp-internals.md             # BBR/CUBIC, conntrack, XDP, 커널 튜닝
    ├── database-internals.md        # MVCC, WAL, B-Tree, 복제 내부
    ├── distributed-systems-advanced.md  # Raft 내부, Consistent Hashing, HLC
    ├── container-internals.md       # Namespace, cgroup v2, Overlay FS, seccomp
    └── performance-engineering.md   # Flame Graph, perf, 레이턴시 분석
```

## 학습 우선순위

1. **Linux & OS** — 모든 인프라의 기반
2. **Network** — 트래픽 흐름과 장애 원인 파악에 필수
3. **Distributed Systems** — SRE의 핵심 도메인
4. **Observability** — 장애 감지 및 대응
5. **Database** — 성능 병목 원인 분석
6. **JVM** — Java/Tomcat WAS 운영 필수
7. **Security** — 인프라 보안 hardening
8. **System Design** — 아키텍처 설계 및 리뷰

## 학습 단계

| 단계 | 디렉토리 | 대상 |
|------|---------|------|
| 기초~중급 | `01_os` ~ `09_container` | 주니어~미드레벨 |
| 심화 | `deepdive/` | Senior SRE/DevOps |

## 이 저장소 사용 방법
- 각 파일은 개념 설명 + 실무 적용 포인트로 구성됩니다.
- `# 실무 포인트` / `# 실무 심화 포인트` 섹션은 면접 및 실제 업무에서 바로 쓸 수 있는 내용입니다.
- Claude에게 특정 주제 심화 내용 추가를 요청할 수 있습니다.
