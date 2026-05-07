# CS Basic — 프로젝트 가이드

## 목적
CS 기초 지식을 체계적으로 정리한 저장소입니다. SRE/DevOps 실무와 연결되는 CS 이론을 한국어로 설명합니다.

---

## 디렉토리 구조

```
cs-basic/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── README.md                  # 전체 문서 목록 & 학습 가이드
├── .claude/
│   ├── settings.json          # 권한 설정 + PostToolUse 훅
│   └── commands/              # 커스텀 슬래시 명령어
│       ├── new-doc.md         # /new-doc — 새 CS 개념 문서 생성
│       ├── new-runbook.md     # /new-runbook — 새 기술 분석 런북 생성
│       ├── review-doc.md      # /review-doc — 문서 품질 검토
│       ├── add-troubleshooting.md  # /add-troubleshooting — 장애 분석 추가
│       └── search-kb.md       # /search-kb — 지식베이스 검색
├── agents/                    # 전문 에이전트 정의
│   ├── doc-writer.md          # CS 문서 작성 전문가
│   ├── interview-coach.md     # 기술 면접 코치
│   ├── deepdive-advisor.md    # 심화 개념 분석가
│   └── system-designer.md     # 시스템 설계 전문가
├── templates/                 # 문서 템플릿
│   ├── service-doc.md         # CS 개념 문서 템플릿
│   ├── runbook.md             # 기술 분석 런북 템플릿
│   └── incident-report.md     # 장애 분석 보고서 템플릿
├── rules/                     # Claude 작성 규칙
│   ├── doc-writing.md         # 문서 작성 원칙
│   ├── cs-conventions.md      # CS 문서 표준 관행
│   ├── security-checklist.md  # 보안 개념 체크리스트
│   └── monitoring.md          # 모니터링 개념 지침
├── 01_os/                     # 운영체제 (5개 파일)
├── 02_network/                # 네트워킹 (9개 파일)
├── 03_database/               # 데이터베이스 (3개 파일)
├── 04_distributed-systems/    # 분산 시스템 (2개 파일)
├── 05_system-design/          # 시스템 설계 (4개 파일)
├── 06_security/               # 보안 (2개 파일)
├── 07_observability/          # 모니터링 & 관찰가능성 (1개 파일)
├── 08_jvm/                    # JVM 기초 (3개 파일)
├── 09_container/              # 컨테이너 & 쿠버네티스 (2개 파일)
└── deepdive/                  # Senior 레벨 심화 (6개 파일)
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 CS 개념 문서 생성 | `/new-doc 01_os/linux-signals` |
| `/new-runbook` | 새 기술 분석 런북 생성 | `/new-runbook TCP TIME_WAIT 분석` |
| `/review-doc` | 문서 품질 검토 | `/review-doc 02_network/http-https.md` |
| `/add-troubleshooting` | 장애 분석 케이스 추가 | `/add-troubleshooting OOM killer` |
| `/search-kb` | 지식베이스 검색 | `/search-kb MVCC 동시성` |

---

## 학습 우선순위

1. **Linux & OS** — 모든 인프라의 기반
2. **Network** — 트래픽 흐름과 장애 원인 파악에 필수
3. **Distributed Systems** — SRE의 핵심 도메인
4. **Observability** — 장애 감지 및 대응
5. **Database** — 성능 병목 원인 분석
6. **JVM** — Java/Tomcat WAS 운영 필수
7. **Security** — 인프라 보안 hardening
8. **System Design** — 아키텍처 설계 및 리뷰

---

## 학습 단계

| 단계 | 디렉토리 | 대상 |
|------|---------|------|
| 기초~중급 | `01_os` ~ `09_container` | 주니어~미드레벨 |
| 심화 | `deepdive/` | Senior SRE/DevOps |

---

## 문서 목록

### 01_os/
| 파일 | 주제 |
|---|---|
| `linux-fundamentals.md` | 명령어, 진단 도구 (strace/lsof/iostat) |
| `process-thread.md` | PCB, 스케줄링 알고리즘, 동기화, Thread Dump |
| `memory-management.md` | 가상 메모리, 페이지 교체 알고리즘, Thrashing |
| `fd-io-models.md` | FD, Blocking/Non-Blocking/epoll, C10K |
| `syscall-kernel.md` | 시스템 콜, 커널/유저 모드, strace |

### 02_network/
| 파일 | 주제 |
|---|---|
| `osi-tcpip.md` | OSI/TCP, TCP 혼잡제어 (CWND/Slow Start) |
| `dns.md` | DNS 쿼리 흐름, TTL, 레코드 타입 |
| `http-https.md` | HTTP/1.1~3, QUIC, TLS, SNI |
| `load-balancing.md` | LB 알고리즘 상세 (RR, LC, IP Hash 등) |
| `socket-port.md` | 소켓 API, 백로그, 소켓 옵션 튜닝 |
| `iptables-nat.md` | Netfilter, iptables, NAT, conntrack |
| `cdn.md` | CDN 동작 원리, Edge, 캐싱 전략 |
| `vpn.md` | VPN 터널링, IPsec, WireGuard |
| `advanced-networking.md` | BGP, OSPF, ECMP, SR-IOV, DPDK (심화) |

### 03_database/
| 파일 | 주제 |
|---|---|
| `sql-basics.md` | ACID, 격리 수준, 인덱스, EXPLAIN |
| `transaction-lock.md` | WAL, 락 종류, 갭락, 데드락, MVCC |
| `nosql-caching.md` | Redis, MongoDB, 캐싱 패턴 |

### 04_distributed-systems/
| 파일 | 주제 |
|---|---|
| `cap-theorem.md` | CAP 정리, BASE vs ACID, 일관성 모델, 복제 |
| `messaging.md` | Kafka, 메시지 큐, Pub/Sub, 이벤트 스트리밍 |

### 05_system-design/
| 파일 | 주제 |
|---|---|
| `scalability.md` | 수평/수직 확장, 샤딩, 파티셔닝, Auto Scaling |
| `reliability.md` | SLI/SLO/SLA, 에러 버짓, Circuit Breaker, 장애 설계 |
| `cicd-deployment.md` | CI/CD 파이프라인, Blue-Green/Canary/Rolling 배포 |
| `backup.md` | 파일 백업 vs 스냅샷 백업, RTO/RPO, 백업 전략 |

### 06_security/
| 파일 | 주제 |
|---|---|
| `security-basics.md` | 보안 기초, 위협 모델링 |
| `auth.md` | JWT 심화, OAuth 2.0, PKCE, RBAC |

### 07_observability/
| 파일 | 주제 |
|---|---|
| `monitoring-logging.md` | 메트릭, 로그, 트레이싱, OpenTelemetry |

### 08_jvm/
| 파일 | 주제 |
|---|---|
| `jvm-architecture.md` | 클래스 로더, 실행 엔진, JIT 컴파일러 |
| `jvm-memory.md` | Heap/Stack/Metaspace/Native Memory |
| `gc-basics.md` | GC 알고리즘, GC 종류, 튜닝 기초 |

### 09_container/
| 파일 | 주제 |
|---|---|
| `docker-basics.md` | Namespace/cgroup/OverlayFS, 네트워킹 |
| `kubernetes-basics.md` | 아키텍처, 스케줄링, Service/RBAC/NetworkPolicy |

### deepdive/
| 파일 | 주제 |
|---|---|
| `linux-kernel-internals.md` | eBPF, syscall, NUMA, io_uring |
| `tcp-internals.md` | BBR/CUBIC, conntrack, XDP, 커널 튜닝 |
| `tcp-deep-dive.md` | TCP 상태머신, SYN/ACK/4-way 커널 내부, 흐름제어/혼잡제어 심화, VPC Flow Log, sysctl 완전 정리 |
| `database-internals.md` | MVCC, WAL, B-Tree, 복제 내부 |
| `distributed-systems-advanced.md` | Raft 내부, Consistent Hashing, HLC |
| `container-internals.md` | Namespace, cgroup v2, Overlay FS, seccomp |
| `performance-engineering.md` | Flame Graph, perf, 레이턴시 분석 |

---

## 이 저장소 사용 방법
- 각 파일은 개념 설명 + 실무 적용 포인트로 구성됩니다.
- `# 실무 포인트` / `# 실무 심화 포인트` 섹션은 면접 및 실제 업무에서 바로 쓸 수 있는 내용입니다.
- Claude에게 특정 주제 심화 내용 추가를 요청할 수 있습니다.
