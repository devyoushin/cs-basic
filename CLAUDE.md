# CS Basic

## 목적
CS 기초 지식을 체계적으로 정리한 저장소입니다.

## 디렉토리 구조

```
cs-basic/
├── CLAUDE.md                  # 이 파일 (프로젝트 가이드)
├── 01_os/                     # 운영체제
│   ├── linux-fundamentals.md
│   ├── process-thread.md
│   └── memory-management.md
├── 02_network/                # 네트워킹
│   ├── osi-tcpip.md
│   ├── dns.md
│   ├── http-https.md
│   └── load-balancing.md
├── 03_database/               # 데이터베이스
│   ├── sql-basics.md
│   ├── indexing-optimization.md
│   └── nosql-caching.md
├── 04_distributed-systems/    # 분산 시스템
│   ├── cap-theorem.md
│   ├── consensus.md
│   └── messaging.md
├── 05_system-design/          # 시스템 설계
│   ├── scalability.md
│   └── reliability.md
├── 06_security/               # 보안
│   └── security-basics.md
├── 07_observability/          # 모니터링 & 관찰가능성
│   └── monitoring-logging.md
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
6. **Security** — 인프라 보안 hardening
7. **System Design** — 아키텍처 설계 및 리뷰

## 학습 단계

| 단계 | 디렉토리 | 대상 |
|------|---------|------|
| 기초~중급 | `01_os` ~ `07_observability` | 주니어~미드레벨 |
| 심화 | `deepdive/` | Senior SRE/DevOps |

## 이 저장소 사용 방법
- 각 파일은 개념 설명 + 실무 적용 포인트로 구성됩니다.
- `# 실무 포인트` / `# 실무 심화 포인트` 섹션은 면접 및 실제 업무에서 바로 쓸 수 있는 내용입니다.
- Claude에게 특정 주제 심화 내용 추가를 요청할 수 있습니다.
