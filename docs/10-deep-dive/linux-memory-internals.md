# Linux 메모리 내부 동작 완전 분석 (Senior Level)

> 가상 메모리, 페이지 폴트, NUMA, HugePage, OOM Killer, cgroup 메모리 컨트롤러

---

## 1. 가상 메모리 주소 공간

### 프로세스 메모리 레이아웃

```
64비트 Linux 프로세스 주소 공간 (상위 → 하위):

0xFFFFFFFFFFFFFFFF ┌───────────────────────────────┐
                   │   커널 공간 (128TB)             │ ← 프로세스 접근 불가
0xFFFF800000000000 ├───────────────────────────────┤
                   │   (비표준 영역, 접근 불가)       │
0x00007FFFFFFFFFFF ├───────────────────────────────┤
                   │   스택 (아래로 증가)             │ ← argc, argv, 환경변수
                   │   ...                          │
                   ├───────────────────────────────┤
                   │   mmap 영역 (공유 라이브러리,   │ ← 아래로 성장
                   │   익명 mmap, 파일 매핑)          │
                   ├───────────────────────────────┤
                   │   힙 (위로 증가)                │ ← malloc/brk
                   ├───────────────────────────────┤
                   │   BSS (초기화 안 된 전역 변수)  │
                   ├───────────────────────────────┤
                   │   데이터 (.data, 초기화된 전역) │
                   ├───────────────────────────────┤
                   │   텍스트 (.text, 코드)          │
0x0000000000400000 └───────────────────────────────┘
```

```bash
# 프로세스 메모리 맵 확인
cat /proc/<pid>/maps
# 또는 상세 통계
cat /proc/<pid>/smaps
# pmap으로 요약
pmap -x <pid>

# 전체 메모리 사용 현황
cat /proc/meminfo
free -h
```

### 가상 주소 → 물리 주소 변환 (4단계 페이지 테이블)

```
64비트 Linux (4KB 페이지 기준):
가상 주소 48비트 = [PGD 9bit][PUD 9bit][PMD 9bit][PTE 9bit][Offset 12bit]

CPU (MMU)가 TLB miss 시:
  1. CR3 레지스터 → PGD(Page Global Directory) 기주소
  2. 가상주소[47:39] → PGD 엔트리 → PUD 기주소
  3. 가상주소[38:30] → PUD 엔트리 → PMD 기주소
  4. 가상주소[29:21] → PMD 엔트리 → PTE 기주소
  5. 가상주소[20:12] → PTE 엔트리 → 물리 프레임 번호(PFN)
  6. PFN << 12 + Offset = 물리 주소

TLB 히트: 변환 캐시 사용 → 즉시 물리 주소
TLB 미스: 페이지 테이블 순회 (hw 또는 sw walk)
TLB 무효화(flush): 컨텍스트 스위치, mmap/munmap 시

KPTI (Kernel Page Table Isolation):
  Meltdown 취약점 완화
  유저 공간: 커널 페이지 테이블 항목 제거 → 컨텍스트 스위치 비용 증가
```

---

## 2. 페이지 폴트 처리

### 페이지 폴트 유형

```
Minor Fault (소프트 폴트):
  페이지가 메모리에 있지만 페이지 테이블에 매핑 안 됨
  → 페이지 테이블 항목만 추가
  → I/O 없음. 빠름 (수 µs)
  예: malloc 후 첫 접근, 공유 라이브러리 로드 (이미 캐시됨)

Major Fault (하드 폴트):
  페이지가 디스크에 있음 (swap 또는 파일)
  → 디스크 I/O 발생 → 페이지 로드 → 테이블 매핑
  → 느림 (수 ms ~ 수백 ms)
  예: swap에서 페이지 복구, 처음 로드하는 파일 매핑

Invalid Fault:
  접근 불가 주소 → SIGSEGV 시그널 전달
  예: NULL dereference, 스택 오버플로, use-after-free
```

```bash
# 프로세스의 페이지 폴트 통계
cat /proc/<pid>/stat | awk '{print "minor:", $10, "major:", $12}'

# 실시간 페이지 폴트 모니터링
perf stat -e page-faults,major-faults sleep 10

# 시스템 전체 페이지 폴트
vmstat 1
# pgfault: minor, pgmajfault: major
```

### Copy-on-Write (COW) 메커니즘

```
fork() 동작:
  1. 부모의 페이지 테이블 복사 (물리 메모리 복사 없음)
  2. 양쪽 페이지 테이블의 모든 쓰기 가능 페이지 → Read-Only 표시
  3. 자식이 페이지 수정 시도 → 페이지 폴트 발생
  4. 커널이 해당 페이지 복사 → 자식에게 새 물리 페이지 할당
  5. 페이지 테이블 업데이트 → 쓰기 재시도

최적화:
  exec() 직후 fork() (서버 자식 프로세스) → COW 실제 발생 적음
  Python/Ruby 등 인터프리터: GC로 인한 내부 상태 수정 → COW 많이 발생
  → RSS 증가, 메모리 공유 효율 하락

Redis fork() 시 COW:
  BGSAVE 중 쓰기 요청 많으면 → 대량 COW
  → 메모리 사용량 급증 (최악 2배)
  → vm.overcommit_memory=1 설정 이유
```

---

## 3. 메모리 할당 내부

### 커널 메모리 할당자

```
Buddy System (물리 페이지 할당):
  2^n 크기의 연속 페이지 블록 관리
  free list: order 0(4KB), 1(8KB), 2(16KB) ... 10(4MB)
  할당: 요청 크기 이상의 최소 order 블록 분할
  해제: 인접 buddy 블록이 free이면 병합

  단편화 문제:
    외부 단편화: 연속 메모리 부족 (Buddy가 관리)
    내부 단편화: 할당된 블록 내 낭비 (Slab이 관리)

Slab/SLUB 할당자 (커널 오브젝트용):
  Buddy에서 할당받은 페이지를 동일 크기 오브젝트로 분할
  자주 쓰는 구조체(task_struct, inode, dentry 등) 캐시
  할당/해제 빈도 높아도 단편화 없음
```

### glibc malloc (ptmalloc2)

```
사용자 공간 메모리 할당:
  32바이트 이하: chunk 재사용 (fastbin)
  512바이트 이하: smallbin
  이상: largebin (크기별 정렬)
  128KB 초과: mmap() 직접 요청 (M_MMAP_THRESHOLD)

Arena:
  멀티스레드 경쟁 방지를 위해 스레드별 Arena
  기본 최대 8 × CPU 수 arenas
  Arena 고갈 시 잠금 경쟁 발생

메모리 단편화 문제:
  장시간 실행 서버에서 "메모리 누수처럼 보이는 단편화"
  해결: jemalloc, tcmalloc 으로 교체 (Redis, Firefox 등 사용)
```

```bash
# malloc 통계 (glibc)
cat /proc/<pid>/status | grep -i vm
# VmRSS: 물리 메모리 사용량
# VmPeak: 최대 가상 메모리

# malloc 단편화 확인
malloc_stats() 함수 호출 또는 valgrind --tool=massif
```

---

## 4. 페이지 교체와 스왑

### LRU 페이지 교체 (실제 구현)

```
Linux는 두 개의 LRU 리스트:
  Active List: 최근 접근된 페이지 (유지 대상)
  Inactive List: 오래된 페이지 (교체 대상)

페이지 이동 흐름:
  첫 접근 → Inactive List
  두 번째 접근(referenced bit) → Active List
  Active List 가득 참 → 오래된 페이지 → Inactive List 강등
  메모리 부족 → Inactive List에서 교체 (kswapd 데몬)

File-backed vs Anonymous 페이지:
  File-backed: 디스크 파일에 대응 (dirty이면 writeback, 아니면 그냥 버림)
  Anonymous: swap 공간에 써야 함 (비용 큼)

swappiness:
  vm.swappiness: 0=익명 페이지 교체 최소화, 100=적극적 스왑
  기본값 60. SSD 있으면 10~20 권장. 메모리 집약 서버는 1~5
```

```bash
# 스왑 사용 확인
free -h
swapon --show
cat /proc/meminfo | grep -i swap

# 스왑 사용 중인 프로세스
for pid in /proc/[0-9]*/status; do
  awk '/VmSwap/{if($2>0) print FILENAME, $0}' $pid
done | sort -k3 -rn | head -20

# swappiness 설정
sysctl vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.d/99-memory.conf

# 페이지 캐시 강제 해제 (주의: 일시적 성능 저하)
echo 3 > /proc/sys/vm/drop_caches   # 1=pagecache, 2=slab, 3=둘 다
```

### 메모리 과할당 (Overcommit)

```
vm.overcommit_memory:
  0 (기본): 휴리스틱 검사. 명백한 초과할당만 거부
  1: 항상 허용 (Redis, Java 힙 예약에 사용)
  2: 엄격. swap + (물리메모리 × overcommit_ratio%) 초과 시 거부

가상 메모리 ≠ 물리 메모리:
  malloc(1GB) 성공 → 실제로 페이지 접근 시 물리 할당
  → 순간 물리 메모리 부족 → OOM Killer 발동

vm.overcommit_ratio: 기본 50 (물리메모리의 50%만 overcommit 허용)
```

---

## 5. OOM Killer

### OOM Score 계산

```
각 프로세스에 oom_score 부여 (0~1000):
  기본: 프로세스의 RSS + 페이지 테이블 + swap 사용량 / 총 메모리 × 1000
  조정: oom_score_adj (-1000 ~ +1000)

oom_score_adj 설정:
  -1000: OOM Kill 완전 면제 (시스템 데몬용)
  -500 ~ -100: 낮은 우선순위 (중요 서비스)
  0: 기본값
  +500 ~ +1000: 높은 우선순위 (먼저 kill)

Kubernetes QoS와 oom_score_adj:
  Guaranteed: oom_score_adj = -997
  Burstable: 요청 비율 기반 계산 (낮을수록 보호)
  BestEffort: oom_score_adj = 1000 (가장 먼저 kill)
```

```bash
# 프로세스의 OOM Score 확인
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj

# OOM Score 높은 순 정렬
for pid in /proc/[0-9]*/oom_score; do
  score=$(cat $pid 2>/dev/null)
  [ "$score" -gt 0 ] 2>/dev/null && echo "$score $pid"
done | sort -rn | head -10

# 중요 프로세스 OOM 보호
echo -1000 > /proc/<pid>/oom_score_adj
# 영구적으로 (systemd 서비스)
# [Service] OOMScoreAdjust=-1000

# OOM 이벤트 확인
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom
```

### OOM Kill 발생 시 커널 로그 분석

```
예시 dmesg 출력:
  [12345.678] Out of memory: Kill process 1234 (java) score 890 or sacrifice child
  [12345.679] Killed process 1234 (java) total-vm:4194304kB, anon-rss:3145728kB, file-rss:65536kB

분석:
  score 890: oom_score 값
  total-vm: 가상 메모리 총량
  anon-rss: 익명 페이지 (힙, 스택) 물리 메모리
  file-rss: 파일 매핑 물리 메모리

OOM Kill 전 메모리 덤프:
  [12345.670] Mem-Info: (전체 메모리 상태)
  [12345.671] Node 0 active_anon:... inactive_anon:... active_file:... inactive_file:...
  → active/inactive 비율로 메모리 압박 정도 파악
```

---

## 6. NUMA (Non-Uniform Memory Access)

### NUMA 아키텍처

```
멀티소켓 서버:
  CPU0 ──── Local Memory 0 (빠름, ~80ns)
    │
  QPI/UPI 인터커넥트 (느림, ~160ns)
    │
  CPU1 ──── Local Memory 1

NUMA 노드: CPU + 로컬 메모리의 쌍
  numactl --hardware 로 확인

메모리 접근 비용:
  Local: 1× (기준)
  Remote: 1.5~2× (인터커넥트 경유)

NUMA의 영향:
  프로세스가 NUMA 노드 0에서 실행되는데 메모리가 노드 1에 있으면
  → 모든 메모리 접근이 인터커넥트 경유 → 레이턴시/대역폭 하락
```

```bash
# NUMA 토폴로지 확인
numactl --hardware
numastat

# NUMA 노드별 메모리 현황
numastat -m

# 프로세스의 NUMA 메모리 위치
numastat -p <pid>

# NUMA 지역성 강제 (특정 노드에서만 메모리 할당)
numactl --cpunodebind=0 --membind=0 <command>

# 인터리브 모드 (NUMA 균등 분산)
numactl --interleave=all <command>

# NUMA 불균형 통계
cat /proc/vmstat | grep numa
# numa_local: 로컬 할당
# numa_foreign: 원격 할당 (이 값이 크면 NUMA 불균형)
```

### NUMA 최적화 전략

```
전략 1: 프로세스-메모리 동일 노드 바인딩
  numactl --cpunodebind=0 --membind=0 ./server
  → 단일 노드 메모리 한계 주의

전략 2: 인터리브 (대용량 데이터, 균등 분산)
  numactl --interleave=all ./redis-server
  → Redis, in-memory DB에 적합

전략 3: AutoNUMA (커널 자동 최적화)
  /proc/sys/kernel/numa_balancing = 1
  → 핫 페이지를 접근하는 CPU 노드로 자동 이동
  → 이주 비용 있음, 실시간 워크로드에는 비활성화 권장

전략 4: 트랜스페어런트 HugePage + NUMA
  THP와 NUMA 자동 밸런싱 함께 쓰면 충돌 가능
  → DB 서버: THP 끄고 NUMA 바인딩 사용
```

---

## 7. HugePage

### 왜 HugePage가 필요한가?

```
일반 페이지 (4KB):
  1GB 데이터 = 262,144개 페이지 테이블 항목
  TLB 항목 수 제한 (~1,500개 L1 TLB)
  → TLB miss 빈번 → 페이지 테이블 순회 오버헤드

HugePage (2MB):
  1GB = 512개 항목 → TLB miss 대폭 감소
  TLB 커버리지: 4KB × 1500 = 6MB vs 2MB × 32 = 64MB

1GB HugePage (PDPE1GB):
  DB 버퍼풀, 대용량 캐시에 효과적
  동적 크기 조정 불가 (부팅 시 예약)
```

### Transparent HugePage (THP)

```
THP: 커널이 자동으로 HugePage 사용 결정

모드:
  always: 항상 HugePage 사용 시도
  madvise: madvise(MADV_HUGEPAGE) 힌트 영역만
  never: THP 비활성화

THP의 문제점:
  Compaction: 메모리를 연속 2MB로 정리하는 작업
  → stall 발생 → 레이턴시 스파이크
  → MySQL, Redis, MongoDB 공식 문서: THP never 권장

THP defragmentation:
  /sys/kernel/mm/transparent_hugepage/defrag
  defer+madvise 권장 (즉시 compaction 방지)
```

```bash
# THP 상태 확인
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never → 현재 always

# THP 비활성화 (DB 서버)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 영구 적용 (grub 커맨드라인)
# transparent_hugepage=never

# Static HugePage 설정
sysctl -w vm.nr_hugepages=1024      # 2MB HugePage 1024개 = 2GB
sysctl -w vm.nr_overcommit_hugepages=256  # 부족 시 추가 할당

# HugePage 사용 현황
cat /proc/meminfo | grep Huge
# HugePages_Total: 1024
# HugePages_Free: 512
# HugePages_Rsvd: 256 (예약됨, 아직 미사용)

# 1GB HugePage (부팅 시 예약)
# grub: default_hugepagesz=1G hugepagesz=1G hugepages=16

# Oracle DB / PostgreSQL HugePage 사용 (shared_memory 경유)
grep -i huge /proc/<postgres_pid>/smaps
```

---

## 8. cgroup 메모리 컨트롤러

### cgroup v2 메모리 파일

```
/sys/fs/cgroup/<cgroup>/
  memory.current        ← 현재 사용량 (bytes)
  memory.max            ← 하드 한계 (초과 시 OOM Kill)
  memory.high           ← 소프트 한계 (초과 시 스로틀)
  memory.min            ← 보장량 (메모리 압박 시 보호)
  memory.low            ← 소프트 보장 (최대한 보호)
  memory.swap.max       ← swap 사용 한계
  memory.events         ← 이벤트 카운터 (oom, max 등)
  memory.stat           ← 상세 통계
```

```bash
# 컨테이너 메모리 cgroup (Kubernetes)
CONTAINER_ID=$(docker inspect <container> --format '{{.Id}}')
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

cat ${CGROUP_PATH}/memory.current    # 현재 사용량
cat ${CGROUP_PATH}/memory.max        # limits.memory
cat ${CGROUP_PATH}/memory.events     # oom_kill 횟수 확인

# memory.stat 분석
cat ${CGROUP_PATH}/memory.stat
# anon: 익명 페이지 (힙, 스택)
# file: 파일 캐시
# shmem: 공유 메모리
# pgfault, pgmajfault: 페이지 폴트 수
# oom_kill: OOM Kill 발생 횟수

# memory.events로 OOM 감지
inotifywait -m ${CGROUP_PATH}/memory.events
```

### Memory Pressure와 PSI

```
PSI (Pressure Stall Information): Linux 4.20+
  메모리/CPU/IO 압박을 % 단위로 정량화

cat /proc/pressure/memory
# some avg10=0.00 avg60=0.12 avg300=0.05 total=1234567
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

some: 일부 태스크가 메모리 대기 중인 시간 비율
full: 모든 태스크가 메모리 대기 중인 시간 비율
avg10/60/300: 10초/60초/300초 이동 평균

임계값 알림 (cgroup v2):
echo "some 70 1000000" > /sys/fs/cgroup/myapp/memory.pressure_level
→ 10초 내 70% 압박 시 알림 (Kubernetes Eviction에 활용)
```

---

## 9. 메모리 진단 실전

### 메모리 누수 디버깅

```bash
# Valgrind (개발 환경)
valgrind --leak-check=full --track-origins=yes ./app

# AddressSanitizer (컴파일 시)
gcc -fsanitize=address -g ./app.c -o app

# pmap으로 메모리 세그먼트 추적 (주기적 스냅샷)
pmap -x <pid> > memory_$(date +%s).txt
diff memory_t1.txt memory_t2.txt

# smaps_rollup으로 요약
cat /proc/<pid>/smaps_rollup
# Pss: 공유 페이지 지분 포함한 실제 사용량

# 메모리 사용량 지속 증가하는 프로세스
watch -n5 "ps aux --sort=-%mem | head -20"
```

### 시스템 메모리 현황 해석

```bash
free -h 출력 해석:
              total   used    free  shared  buff/cache  available
Mem:           62Gi   45Gi   1.2Gi  2.1Gi      16Gi       15Gi

available:
  실제로 앱에 즉시 할당 가능한 메모리
  = free + 재사용 가능한 buff/cache
  → 이 값이 중요 (free만 보면 오해)

buff/cache:
  buff: 블록 디바이스 I/O 버퍼
  cache: 파일 시스템 페이지 캐시
  → 메모리 압박 시 커널이 자동으로 해제함 (reclaimable)

/proc/meminfo 핵심 필드:
  MemAvailable: 위의 available
  Dirty: 디스크에 아직 안 쓴 페이지
  Writeback: 현재 디스크에 쓰는 중인 페이지
  Slab: 커널 slab 할당자 사용량
  SReclaimable: 회수 가능한 slab (파일 캐시 inode 등)
  SUnreclaim: 회수 불가 slab (커널 데이터 구조)
  KernelStack: 스레드별 커널 스택 (스레드 많으면 큼)
```

### 페이지 캐시 관련

```bash
# 파일별 페이지 캐시 점유 확인
fincore /path/to/largefile    # fincore 도구 필요

# vmtouch로 파일 캐시 상태 확인/제어
vmtouch -v /path/to/file      # 캐시 상태 확인
vmtouch -l /path/to/file      # 캐시에 고정 (mlock)
vmtouch -e /path/to/file      # 캐시에서 퇴출

# dirty 페이지 쓰기 정책
sysctl vm.dirty_ratio=20          # 전체 메모리의 20% dirty → 강제 writeback
sysctl vm.dirty_background_ratio=10  # 10% → 백그라운드 writeback 시작
sysctl vm.dirty_expire_centisecs=3000 # 30초 후 dirty 페이지 flush
```

---

## 10. 실무 심화 포인트 & 면접 대비

### "메모리가 충분한데 OOM이 발생하는 이유?"
```
가능한 원인:
  1. 연속 물리 메모리 부족 (Buddy System 단편화)
     → DMA 영역(32비트 주소), HugePage 할당 실패
  2. cgroup 메모리 한계 도달 (컨테이너)
     → 노드 메모리는 여유있어도 컨테이너 limits 초과
  3. KernelStack 고갈 (스레드 수 × 8KB 스택)
  4. NUMA 노드 편향 (한 노드 고갈, 다른 노드 여유)
  5. vm.overcommit_memory=2에서 overcommit 한계 도달
```

### "Redis에서 vm.overcommit_memory=1이 필요한 이유?"
```
Redis BGSAVE/BGREWRITEAOF:
  fork() 호출 → 자식 프로세스가 RDB/AOF 저장
  fork() 시 가상 메모리 전체 예약 필요 (COW 방식이지만 예약은 필요)
  → vm.overcommit_memory=0에서 fork() 실패 가능
  → "Cannot allocate memory" 에러

권장 설정:
  vm.overcommit_memory=1
  vm.swappiness=0 또는 1
  transparent_hugepage=never
```

### "THP를 끄는 이유?"
```
THP 문제:
  Compaction: 연속 2MB 메모리 확보 위한 페이지 이동
  → 수백 ms 레이턴시 스파이크 → p99 레이턴시 급등

영향받는 시스템:
  MySQL/MariaDB: 자체 버퍼풀 관리, THP 불필요
  Redis: 위 참조
  MongoDB: 명시적으로 never 권장
  Java: G1GC와 THP 상호작용으로 pause 증가 가능

DB 서버 기본 설정:
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag
```

### "NUMA를 고려해야 하는 상황은?"
```
고려 필요한 경우:
  멀티소켓 서버 (2 소켓 이상)
  레이턴시 민감한 워크로드 (DB, 캐시)
  메모리 대역폭 집약적 처리

확인 방법:
  numastat → numa_foreign 값이 크면 NUMA 불균형
  perf stat -e cache-misses,LLC-misses → 캐시 미스율

대응:
  단일 노드 바인딩: numactl --membind=0 --cpunodebind=0
  Interleave: 대용량 균등 분산
  AutoNUMA: 자동 최적화 (레이턴시 민감하지 않은 경우)
```

| 질문 | 핵심 답변 |
|------|-----------|
| Minor vs Major fault | Minor=메모리에 있음(I/O 없음), Major=디스크에서 로드(I/O 발생) |
| COW의 의미 | fork() 후 쓸 때만 물리 메모리 복사. 공유 읽기는 같은 물리 페이지 |
| Slab 할당자 목적 | 커널 오브젝트 재사용. Buddy의 내부 단편화 해결 |
| available vs free | free=완전 비어있는 메모리, available=캐시 해제 포함 할당 가능량 |
| OOM Score 결정 요소 | RSS + swap 사용량 비율 + oom_score_adj 조정값 |
| HugePage 효과 | TLB 커버리지 증가 → TLB miss 감소 → 메모리 집약 앱 성능 향상 |
