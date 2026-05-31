# OS 기초 완전 정리

> **카테고리**: 01_os
> **레벨**: 기초~Senior 심화
> **작성일**: 2026-05-11

---

## 목차

1. [메모리 관리](#1-메모리-관리)
2. [프로세스/스레드 동기화](#2-프로세스스레드-동기화)
3. [파일 시스템](#3-파일-시스템)
4. [시스템 콜](#4-시스템-콜)
5. [I/O 모델](#5-io-모델)
6. [CPU 스케줄링](#6-cpu-스케줄링)
7. [네트워크 스택 (커널 내부)](#7-네트워크-스택-커널-내부)
8. [부팅 프로세스 & cgroup](#8-부팅-프로세스--cgroup)

---

## 1. 메모리 관리

### 1.1 가상 메모리 (Virtual Memory)

각 프로세스는 독립된 **가상 주소 공간**을 가진다. 물리 RAM과 1:1 매핑이 아니며, MMU(Memory Management Unit)가 주소 변환을 담당한다.

```
프로세스 가상 주소 공간 (x86-64, 낮은 주소 → 높은 주소)
┌──────────────────────┐ 0x0000000000000000
│   Text (코드)         │ 실행 가능, 읽기 전용
├──────────────────────┤
│   Data (초기화된 전역) │
├──────────────────────┤
│   BSS (미초기화 전역)  │
├──────────────────────┤
│   Heap               │ ↑ 위로 증가 (malloc)
│       ↑              │
│   (빈 공간)           │
│       ↓              │
│   Stack              │ ↓ 아래로 증가 (지역변수, 콜스택)
├──────────────────────┤
│   Kernel Space        │ 유저 접근 불가
└──────────────────────┘ 0xFFFFFFFFFFFFFFFF
```

**핵심 이점**
- 프로세스 간 메모리 격리 (한 프로세스가 다른 프로세스 메모리에 접근 불가)
- 물리 메모리보다 큰 메모리 사용 가능 (Swap 활용)
- COW(Copy-on-Write): `fork()` 시 페이지를 실제 쓰기 전까지 공유

---

### 1.2 페이징 (Paging)

가상 주소 → 물리 주소 변환 메커니즘. 메모리를 고정 크기(보통 4KB) **페이지**로 분할.

```
가상 주소 (48-bit, x86-64)
┌────────┬────────┬────────┬────────┬──────────────┐
│ PGD idx│ PUD idx│ PMD idx│ PTE idx│  Page Offset │
│  9 bit │  9 bit │  9 bit │  9 bit │    12 bit    │
└────────┴────────┴────────┴────────┴──────────────┘
                                                 ↓
              4단계 페이지 테이블 워크 (Page Table Walk)
              CR3 레지스터 → PGD → PUD → PMD → PTE → 물리 주소
```

**페이지 폴트 (Page Fault)**

| 종류 | 원인 | 처리 |
|------|------|------|
| Minor Fault | 페이지가 메모리에 있지만 매핑 안 됨 (COW 등) | 매핑만 추가, 빠름 |
| Major Fault | 페이지가 디스크(Swap)에 있음 | 디스크 I/O 발생, 느림 |
| Invalid Fault | 잘못된 주소 접근 | SIGSEGV 발생, 프로세스 종료 |

```bash
# 페이지 폴트 확인
cat /proc/<pid>/status | grep -E "VmRSS|VmSize|MajFlt|MinFlt"
# majflt: Major Page Fault 횟수 — 높으면 Swap 사용 중
```

---

### 1.3 TLB (Translation Lookaside Buffer)

페이지 테이블 워크는 4번의 메모리 접근이 필요 → 너무 느림.
TLB는 최근 가상→물리 주소 변환 결과를 캐시하는 **CPU 내부 하드웨어**.

```
가상 주소 → [TLB 히트?] → 물리 주소 (1 사이클)
                 ↓ TLB 미스
             Page Table Walk (수백 사이클) → 물리 주소 + TLB 업데이트
```

**TLB Flush**: 컨텍스트 스위칭 시 전체 TLB가 무효화됨 → **컨텍스트 스위칭 비용의 주요 원인**.
ASID(Address Space ID)를 지원하는 CPU는 프로세스별 TLB 엔트리를 구분하여 flush 최소화.

**HugePage (2MB/1GB 페이지)**
- TLB 엔트리 하나로 더 큰 범위 커버 → TLB 미스 감소
- DB(PostgreSQL, Oracle), JVM 등 대용량 메모리 워크로드에서 성능 향상

```bash
# HugePage 설정 확인
cat /proc/meminfo | grep -i huge
# Transparent HugePage 상태
cat /sys/kernel/mm/transparent_hugepage/enabled
```

---

### 1.4 메모리 단편화 (Fragmentation)

| 종류 | 설명 | 영향 |
|------|------|------|
| 외부 단편화 | 전체 여유 공간은 있으나 연속된 블록이 없음 | 큰 메모리 할당 실패 |
| 내부 단편화 | 할당된 블록이 실제 요청보다 큼 | 메모리 낭비 |

**커널 해결책**
- **Buddy System**: 2의 거듭제곱 단위로 블록 분할/병합 → 외부 단편화 완화
- **Slab Allocator**: 자주 쓰는 객체 크기별로 캐시 유지 → 내부 단편화 완화, 할당 속도 향상

```bash
# Slab 캐시 현황
cat /proc/slabinfo
slabtop  # 실시간 slab 사용량
```

---

### 1.5 Swap

RAM이 부족할 때 사용하지 않는 페이지를 디스크(Swap 영역)로 내보내는 메커니즘.

```
RAM 부족 → kswapd 커널 스레드 활성화
         → LRU 리스트에서 오래된 페이지 선택
         → 더티 페이지(수정됨): 디스크에 쓴 후 해제
         → 클린 페이지(읽기 전용): 그냥 해제 (필요 시 파일에서 다시 로드)
         → Swap 페이지: swap 영역에 기록
```

**swappiness**: Swap 사용 적극성 (0~100). SRE 권장값은 워크로드에 따라 다름.

```bash
# 현재 swappiness
cat /proc/sys/vm/swappiness

# 임시 변경
echo 10 > /proc/sys/vm/swappiness

# Swap 사용 현황
free -h
swapon --show
vmstat 1   # si(swap in), so(swap out) 컬럼 확인
```

**Swap Storm**: 메모리 부족 시 연속적인 page-in/page-out으로 CPU가 I/O에 묶이는 현상 (Thrashing).

---

### 1.6 OOM Killer

물리 메모리 + Swap 모두 소진 시 커널이 직접 프로세스를 종료시키는 메커니즘.

**OOM Score 계산 (`/proc/<pid>/oom_score`)**
- RSS가 클수록 점수 높음
- `oom_score_adj` (-1000~1000): 낮을수록 보호, -1000이면 절대 종료 안 함

```bash
# OOM 종료 이력 확인
dmesg | grep -i "oom\|killed"
journalctl -k | grep -i oom

# 중요 프로세스 OOM 보호
echo -500 > /proc/<pid>/oom_score_adj

# 컨테이너/JVM은 oom_score_adj로 우선순위 조정
```

**OOM 발생 패턴**
```
oom-kill: constraint=CONSTRAINT_NONE, nodemask=(null),
          cpuset=/, mems_allowed=0,
          task_memcg=/system.slice/app.service,
          task=java, pid=1234, uid=1000
```

---

### 1.7 malloc 내부 동작

`malloc()`은 libc(glibc) 함수. 커널 syscall인 `brk()` / `mmap()`을 래핑.

```
malloc(size) 호출
    ↓
size < 128KB → brk() 사용 (힙 경계 확장)
size ≥ 128KB → mmap(MAP_ANONYMOUS) 사용 (별도 메모리 영역)
    ↓
free() 호출 시: 바로 OS에 반환하지 않고 glibc 내부 free list에 보관
    → 다음 malloc() 시 재활용 (할당/해제 반복 시 성능 최적화)
```

**메모리 누수 진단**

```bash
# Valgrind: 개발 환경에서 메모리 누수 탐지
valgrind --leak-check=full ./myapp

# /proc/<pid>/smaps: 실제 메모리 사용 상세 정보
cat /proc/<pid>/smaps | grep -E "Rss|Pss|Swap"

# pmap: 프로세스 메모리 매핑
pmap -x <pid>
```

---

## 2. 프로세스/스레드 동기화

### 2.1 Race Condition (경쟁 조건)

둘 이상의 스레드/프로세스가 공유 자원에 동시에 접근하여 실행 순서에 따라 결과가 달라지는 현상.

```c
// Race Condition 예시
int counter = 0;

// Thread 1          // Thread 2
counter++;           counter++;
// LOAD counter (0)  // LOAD counter (0)  ← 동시 실행
// ADD 1             // ADD 1
// STORE 1           // STORE 1           ← 최종값: 1 (예상: 2)
```

**원인**: CPU가 LOAD-MODIFY-STORE를 원자적으로 실행하지 않음 → 인터럽트 또는 스케줄링으로 중간에 선점 가능.

---

### 2.2 Mutex (Mutual Exclusion)

한 번에 하나의 스레드만 임계구역(Critical Section)에 진입하도록 보장하는 락.

```
Mutex 동작:
lock() → 임계구역 진입 → unlock()
         ↑ 다른 스레드: 블로킹 대기 (sleep 상태)

특징:
- 소유권 개념 있음 (lock한 스레드만 unlock 가능)
- 대기 시 스레드 sleep → Context Switch 발생
- 커널 syscall (futex) 사용
```

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);
counter++;  // 임계구역
pthread_mutex_unlock(&lock);
```

---

### 2.3 Spinlock

대기 중 CPU를 계속 점유하며 반복 확인(busy-waiting)하는 락.

```
Spinlock 동작:
while (lock != FREE) { /* CPU 점유하며 대기 */ }
lock = ACQUIRED;
// 임계구역
lock = FREE;

특징:
- Context Switch 없음 → 아주 짧은 임계구역에서 Mutex보다 빠름
- 대기 시간이 길면 CPU 낭비
- 커널 내부, 인터럽트 핸들러에서 주로 사용 (sleep 불가능한 컨텍스트)
- 단일 코어에서는 사용 불가 (대기 중 다른 스레드 실행 못 함)
```

**언제 Mutex vs Spinlock?**

| | Mutex | Spinlock |
|--|-------|----------|
| 임계구역 길이 | 길 때 | 매우 짧을 때 |
| 컨텍스트 | 유저 스페이스 | 커널/인터럽트 핸들러 |
| CPU 낭비 | 없음 (sleep) | 있음 (busy-wait) |

---

### 2.4 Semaphore

N개의 스레드가 동시에 접근 가능하도록 제어하는 카운팅 메커니즘.

```
Binary Semaphore (= Mutex와 유사, 단 소유권 없음)
Counting Semaphore: 동시 접근 수 제한

sem_wait(s):   s-- → s < 0이면 블로킹
sem_post(s):   s++ → 대기 중인 스레드 깨움

예시: DB 커넥션 풀 최대 10개
sem_init(&sem, 0, 10);  // 초기값 10
sem_wait(&sem);          // 커넥션 획득 (10→9→...→0이면 대기)
// DB 사용
sem_post(&sem);          // 커넥션 반환
```

**Mutex vs Semaphore**

| | Mutex | Semaphore |
|--|-------|-----------|
| 소유권 | 있음 (같은 스레드만 해제) | 없음 (다른 스레드가 해제 가능) |
| 용도 | 상호 배제 | 자원 수 제한, 스레드 간 시그널 |
| 초기값 | 1 | N (임의) |

---

### 2.5 Deadlock (교착 상태)

둘 이상의 스레드가 서로 상대방이 가진 락을 기다리며 영원히 블로킹되는 상태.

**4가지 필요 조건 (Coffman 조건)**

| 조건 | 설명 |
|------|------|
| 상호 배제 | 자원은 한 번에 하나만 사용 |
| 점유 대기 | 자원 점유 상태에서 다른 자원 대기 |
| 비선점 | 강제로 자원 빼앗기 불가 |
| 순환 대기 | A→B→C→A 형태의 대기 사이클 |

**예시 & 해결**

```
Thread 1: lock(A) → lock(B)   →  Deadlock
Thread 2: lock(B) → lock(A)

해결: 락 획득 순서 전역 통일 (항상 A 먼저, B 나중)
```

```bash
# Java Thread Dump에서 Deadlock 탐지
jstack <pid> | grep -A 5 "deadlock"

# 커널 레벨 스레드 상태
cat /proc/<pid>/task/<tid>/status | grep State
```

---

## 3. 파일 시스템

### 3.1 inode 구조

파일의 **메타데이터**를 저장하는 자료구조. 파일 이름은 inode에 없음 (디렉터리 엔트리에 있음).

```
inode 구성:
┌─────────────────────────────┐
│ 파일 타입 (일반/디렉터리/링크) │
│ 권한 (rwxrwxrwx)             │
│ UID, GID                    │
│ 파일 크기                    │
│ 타임스탬프 (atime/mtime/ctime)│
│ 링크 카운트                  │
│ 데이터 블록 포인터 →          │──→ 실제 데이터 블록
│   Direct (12개)              │
│   Indirect (1단계)           │
│   Double Indirect (2단계)    │
│   Triple Indirect (3단계)    │
└─────────────────────────────┘
```

**디렉터리 구조**
```
디렉터리 = inode번호 + 파일이름의 매핑 테이블

/home/sunny/
├── inode 1234 → "app.py"
├── inode 5678 → "config.yaml"
└── inode 9999 → ".bashrc"
```

```bash
# inode 번호 확인
ls -i /home/sunny/app.py

# inode 사용량 (inode 고갈 → "No space left" 에러)
df -i

# 파일 상세 메타데이터
stat /home/sunny/app.py
```

**Hard Link vs Symbolic Link**

| | Hard Link | Symbolic Link |
|--|-----------|--------------|
| 동작 | 같은 inode를 가리키는 다른 이름 | 경로를 저장하는 별도 파일 |
| 원본 삭제 시 | 데이터 유지 (링크 카운트 감소) | 깨진 링크 (dangling link) |
| 크로스 파티션 | 불가 | 가능 |

---

### 3.2 VFS (Virtual File System)

다양한 파일 시스템(ext4, XFS, tmpfs, NFS...)을 **동일한 인터페이스**로 추상화하는 커널 레이어.

```
유저 스페이스: open("/data/file") → read() → write()
                       ↓
VFS 레이어:    통일된 파일 오퍼레이션 (file_operations 구조체)
                       ↓
실제 FS:       ext4 / XFS / NFS / tmpfs / procfs ...
```

**VFS 핵심 객체**

| 객체 | 역할 |
|------|------|
| superblock | 파일시스템 전체 메타데이터 |
| inode | 파일 메타데이터 |
| dentry | 디렉터리 캐시 (경로 → inode 매핑) |
| file | 열린 파일 인스턴스 (프로세스별 오프셋 등) |

---

### 3.3 ext4 vs XFS

| 특성 | ext4 | XFS |
|------|------|-----|
| 저널링 | 있음 (ordered/writeback/journal 모드) | 있음 (로그 구조) |
| 최대 파일 크기 | 16TB | 8EB |
| 최대 FS 크기 | 1EB | 8EB |
| 소규모 파일 | 빠름 | 보통 |
| 대규모 파일 | 보통 | 매우 빠름 (병렬 I/O) |
| 주 사용처 | 일반 서버, OS 파티션 | 대용량 스토리지, DB |
| 온라인 축소 | 불가 | 불가 |

**저널링 (Journaling)**
```
목적: 비정상 종료 후 파일시스템 일관성 복구

동작:
1. 변경 내용을 Journal(로그)에 먼저 기록 (Write-Ahead Logging)
2. 실제 데이터 블록 업데이트
3. Journal에서 커밋 완료 표시

재부팅 시: 커밋 안 된 Journal 항목 → rollback 또는 재실행
```

---

### 3.4 Page Cache

디스크 I/O 결과를 RAM에 캐시하는 커널 메커니즘. **모든 파일 I/O는 Page Cache를 통한다**.

```
read("/data/file"):
  Page Cache 히트 → RAM에서 직접 반환 (매우 빠름)
  Page Cache 미스 → 디스크 읽기 → Page Cache에 저장 → 반환

write("/data/file"):
  Page Cache에 쓰기 (dirty 표시)
  → 비동기적으로 디스크에 flush (pdflush / writeback 스레드)
  → fsync() 호출 시 즉시 flush 강제
```

```bash
# Page Cache 사용량 확인
free -h   # buff/cache 컬럼

# Page Cache 강제 비우기 (테스트/벤치마킹 시)
echo 3 > /proc/sys/vm/drop_caches

# dirty page 비율 제어
sysctl vm.dirty_ratio          # dirty page가 이 비율 넘으면 write 차단
sysctl vm.dirty_background_ratio  # 백그라운드 flush 시작 비율
```

---

### 3.5 /proc & /sys 가상 파일시스템

**`/proc` (procfs)**: 커널이 메모리에서 실시간으로 생성하는 가상 FS. 프로세스/커널 상태 조회.

```bash
/proc/<pid>/
├── status         # 프로세스 상태, 메모리 사용량
├── maps           # 가상 메모리 매핑
├── fd/            # 열린 파일 디스크립터
├── net/tcp        # TCP 연결 상태
├── cmdline        # 실행 명령어
└── cgroup         # 속한 cgroup

/proc/sys/        # 커널 파라미터 (sysctl)
├── net/ipv4/tcp_fin_timeout
├── vm/swappiness
└── kernel/pid_max
```

**`/sys` (sysfs)**: 커널 객체(드라이버, 디바이스) 계층 구조를 노출.

```bash
/sys/
├── block/sda/queue/scheduler   # I/O 스케줄러 확인/변경
├── class/net/eth0/             # 네트워크 인터페이스 정보
├── kernel/mm/transparent_hugepage/
└── fs/cgroup/                  # cgroup 계층
```

---

## 4. 시스템 콜

### 4.1 유저 모드 vs 커널 모드

```
Ring 레벨 (x86):
┌───────────────────────────────────┐
│ Ring 0 (Kernel Mode)              │ 모든 명령어, 하드웨어 직접 접근
│   OS 커널, 드라이버               │
├───────────────────────────────────┤
│ Ring 3 (User Mode)                │ 제한된 명령어 셋
│   애플리케이션, 라이브러리         │
└───────────────────────────────────┘
```

| | 유저 모드 | 커널 모드 |
|--|----------|---------|
| 접근 권한 | 자기 메모리만 | 모든 메모리/하드웨어 |
| 오류 영향 | 해당 프로세스만 종료 | Kernel Panic |
| 진입 방법 | 시스템 콜, 인터럽트, 예외 | iret 명령으로 복귀 |

---

### 4.2 Syscall 동작 원리

```
유저 프로그램: read(fd, buf, len)
                ↓
glibc:         syscall 번호를 rax 레지스터에 저장
               SYSCALL 명령어 실행
                ↓
CPU:           Ring3 → Ring0 전환
               커널 스택으로 전환
               MSR_LSTAR 레지스터의 핸들러로 점프
                ↓
커널:          sys_read() 실행
               완료 후 sysret 명령어
                ↓
유저 복귀:     결과를 rax 레지스터로 반환
```

**주요 syscall 번호 (x86-64)**

| syscall | 번호 | 설명 |
|---------|------|------|
| read | 0 | 파일 읽기 |
| write | 1 | 파일 쓰기 |
| open | 2 | 파일 열기 |
| close | 3 | FD 닫기 |
| mmap | 9 | 메모리 매핑 |
| brk | 12 | 힙 경계 변경 |
| clone | 56 | 스레드/프로세스 생성 |
| execve | 59 | 프로그램 실행 |

```bash
# 프로세스가 호출하는 syscall 추적
strace -p <pid>
strace -c ./myapp  # syscall별 횟수/시간 집계
```

---

### 4.3 인터럽트 (Interrupt)

**하드웨어 인터럽트**: 외부 장치(NIC, 디스크)가 CPU에 처리 요청.

```
NIC에 패킷 도착
    ↓
NIC → CPU에 IRQ(Interrupt Request) 신호
    ↓
CPU: 현재 실행 중단, 레지스터 저장
    ↓
IDT(Interrupt Descriptor Table)에서 핸들러 주소 조회
    ↓
인터럽트 핸들러 실행 (커널 모드)
    ↓
원래 실행 재개
```

**소프트웨어 인터럽트 (Trap)**: 프로그램이 의도적으로 발생 (syscall의 구 방식: `int 0x80`).

**인터럽트 vs 폴링**

| | 인터럽트 | 폴링 |
|--|---------|------|
| CPU 사용 | 이벤트 시만 처리 | 지속 확인 |
| 응답속도 | 빠름 | 빠름 (짧은 폴링 시) |
| 고속 I/O | 오버헤드 큼 (100GbE 등) | DPDK 등 kernel bypass 방식 |

---

### 4.4 Context Switch 비용

**컨텍스트 스위치**: CPU가 한 프로세스/스레드에서 다른 것으로 전환하는 과정.

```
Context Switch 단계:
1. 현재 프로세스 레지스터 상태 → PCB에 저장
2. 스케줄러가 다음 프로세스 선택
3. 다음 프로세스 PCB → 레지스터 복원
4. CR3 업데이트 (페이지 테이블 변경) → TLB flush
5. 실행 재개
```

**비용 요소**

| 요소 | 비용 | 설명 |
|------|------|------|
| 레지스터 저장/복원 | 낮음 | 수십 레지스터 |
| TLB flush | 높음 | 이후 메모리 접근 느려짐 |
| Cache 오염 | 높음 | L1/L2 캐시가 새 프로세스 데이터로 교체 |
| 스케줄러 실행 | 낮음~중간 | 스케줄링 알고리즘 연산 |

```bash
# Context Switch 횟수 모니터링
vmstat 1         # cs 컬럼
pidstat -w 1     # cswch/s(자발적), nvcswch/s(비자발적)

# 프로세스별 context switch
cat /proc/<pid>/status | grep ctxt
```

---

## 5. I/O 모델

### 5.1 Blocking I/O

```
유저 프로세스: read(fd, buf, len)
                    ↓
              커널: 데이터 올 때까지 프로세스 SLEEP
                    (소켓 수신 버퍼에 데이터 도착 대기)
                    ↓
              데이터 도착 → 버퍼 복사 → 프로세스 WAKE
                    ↓
              read() 반환
```

**특징**: 단순, 직관적. 연결당 하나의 스레드 필요 → 수천 연결 시 스레드 폭발 (C10K 문제).

---

### 5.2 Non-Blocking I/O

```c
// FD를 Non-Blocking으로 설정
fcntl(fd, F_SETFL, O_NONBLOCK);

// 데이터 없으면 즉시 EAGAIN 반환 (블로킹 없음)
n = read(fd, buf, len);
if (n == -1 && errno == EAGAIN) {
    // 나중에 다시 시도
}
```

---

### 5.3 I/O Multiplexing (select/poll/epoll)

여러 FD를 하나의 스레드에서 효율적으로 감시.

```
select/poll:  전체 FD 목록을 매번 커널에 전달 → O(N) 스캔 → FD 수 많으면 느림
epoll:        관심 FD 등록 후 이벤트 발생 시만 알림 → O(1) → 수만 연결도 처리 가능
```

```c
// epoll 사용 패턴
int epfd = epoll_create1(0);

// FD 등록
struct epoll_event ev = { .events = EPOLLIN, .data.fd = sockfd };
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// 이벤트 대기 (여러 FD 동시 감시)
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);
for (int i = 0; i < n; i++) {
    handle(events[i].data.fd);
}
```

**epoll 동작 원리**
- 커널 내부에 레드-블랙 트리로 관심 FD 관리
- 이벤트 발생 시 준비 목록(ready list)에 추가
- `epoll_wait()`은 ready list만 반환 → O(이벤트 수)

**Edge Trigger vs Level Trigger**

| | Level Trigger (기본) | Edge Trigger (EPOLLET) |
|--|---------------------|----------------------|
| 알림 시점 | 데이터 있는 동안 계속 | 새 데이터 도착 순간만 |
| 처리 방식 | 여러 번 read 가능 | 한 번에 모두 읽어야 함 |
| 사용 | 간단한 구현 | Nginx, 고성능 서버 |

---

### 5.4 비동기 I/O (io_uring)

Linux 5.1+. Submission Queue/Completion Queue 링 버퍼를 유저-커널이 공유 → syscall 오버헤드 최소화.

```
유저:  SQE(Submission Queue Entry) 작성 → 커널에 배치 제출
커널:  비동기 I/O 처리 → CQE(Completion Queue Entry) 작성
유저:  CQE 확인 → 결과 처리

장점: syscall 횟수 극적으로 감소, 커널 폴링 모드에서 syscall 0회 가능
```

---

## 6. CPU 스케줄링

### 6.1 스케줄링 목표

| 목표 | 설명 |
|------|------|
| 처리량 최대화 | 단위 시간당 완료 작업 수 |
| 응답 시간 최소화 | 대화형 프로세스의 체감 속도 |
| 공정성 | 모든 프로세스에 CPU 시간 보장 |
| 기아(Starvation) 방지 | 낮은 우선순위도 언젠가 실행 |

---

### 6.2 주요 스케줄링 알고리즘

**FCFS (First Come First Served)**
```
도착 순서대로 실행. 선점 없음.
단점: Convoy Effect — 긴 작업 뒤에 짧은 작업이 오래 기다림
```

**SJF (Shortest Job First)**
```
남은 실행 시간이 가장 짧은 프로세스 선택.
최소 평균 대기시간 보장. 단점: 실행 시간 예측 어려움, 기아 가능.
```

**Round Robin (RR)**
```
각 프로세스에 고정 타임 슬라이스(Quantum) 부여.
타임 슬라이스 소진 시 선점 → 다음 프로세스로.
타임 슬라이스 크기: 너무 작으면 Context Switch 오버헤드, 너무 크면 FCFS에 근접.
```

**Priority Scheduling**
```
우선순위 높은 프로세스 먼저 실행.
Aging으로 기아 방지: 대기 시간이 길수록 우선순위 상승.
```

---

### 6.3 Linux CFS (Completely Fair Scheduler)

Linux 커널 기본 스케줄러 (Linux 2.6.23+).

```
핵심 개념: vruntime (가상 실행 시간)
- 각 프로세스의 실제 CPU 사용 시간을 가중치로 정규화한 값
- 가장 vruntime이 작은 프로세스를 다음 실행 대상으로 선택
- 레드-블랙 트리로 vruntime 정렬 → O(log N) 선택

nice 값 (-20~19):
- nice -20: 가장 높은 우선순위 (CPU를 많이 받음)
- nice 0: 기본값
- nice 19: 가장 낮은 우선순위
```

```bash
# 프로세스 우선순위 확인
ps -eo pid,ni,pri,cmd

# nice 값 변경
nice -n 10 ./myapp     # 시작 시
renice -n 5 -p <pid>   # 실행 중

# CPU 스케줄링 정책 확인
chrt -p <pid>          # SCHED_OTHER/FIFO/RR 등

# CFS 레이턴시 튜닝
sysctl kernel.sched_latency_ns        # 스케줄링 주기
sysctl kernel.sched_min_granularity_ns # 최소 실행 단위
```

---

### 6.4 실시간 스케줄링 (RT)

`SCHED_FIFO`, `SCHED_RR`: 우선순위 기반, CFS보다 항상 먼저 실행.
사용처: 오디오 서버, 산업용 제어, 고빈도 트레이딩.

```bash
# 실시간 정책으로 실행
chrt -f 50 ./latency-critical-app   # SCHED_FIFO, 우선순위 50
```

---

## 7. 네트워크 스택 (커널 내부)

### 7.1 소켓 버퍼 구조

```
NIC 수신
    ↓
DMA → sk_buff (커널 패킷 버퍼) 생성
    ↓
TCP 계층: 수신 버퍼(sk_rcvbuf)에 누적
    ↓
애플리케이션: recv()/read() 호출 시 커널 버퍼 → 유저 버퍼 복사

※ 제로카피(sendfile, splice): 유저 버퍼 복사 생략
```

**소켓 버퍼 크기 튜닝**

```bash
# 현재 소켓 버퍼 크기 (bytes)
sysctl net.core.rmem_max        # 최대 수신 버퍼
sysctl net.core.wmem_max        # 최대 송신 버퍼
sysctl net.ipv4.tcp_rmem        # TCP 수신 버퍼 (min default max)
sysctl net.ipv4.tcp_wmem        # TCP 송신 버퍼

# 고성능 서버 권장 설정
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"

# 특정 연결의 버퍼 사용량
ss -ntp | head -5
# Recv-Q: 수신 버퍼에 쌓인 미읽은 바이트
# Send-Q: 송신 버퍼에서 ACK 안 받은 바이트
```

---

### 7.2 TCP 커널 내부 동작

**3-Way Handshake (커널 관점)**

```
Client                    Server 커널
  │─── SYN ────────────→  SYN_RECV 상태, SYN Queue에 추가
  │←── SYN+ACK ─────────
  │─── ACK ────────────→  ESTABLISHED, Accept Queue로 이동
                           → accept() 호출 시 FD 반환
```

**SYN Backlog 큐 오버플로**
```
SYN Queue  (net.ipv4.tcp_max_syn_backlog): 3-Way 완료 전 연결 대기
Accept Queue (listen() backlog):           완료된 연결 대기 (accept() 호출 전)

큐 오버플로 시: 패킷 drop → 클라이언트 Connection Timeout
```

```bash
# SYN Queue 오버플로 확인
netstat -s | grep -i "syn"
# TcpExtTCPReqQFullDrop: SYN Queue drop 횟수

# Accept Queue 확인
ss -lnt  # Recv-Q: Accept Queue 현재 크기
```

---

### 7.3 TIME_WAIT 대량 발생 원인과 처리

**TIME_WAIT 목적**
```
TCP 4-Way Close 후 활성 종료 측(Active Close)이 2MSL(보통 60초) 동안 대기.
이유: 1) 마지막 ACK 유실 시 FIN 재전송 가능
      2) 이전 연결의 지연 패킷이 새 연결에 섞이지 않도록
```

**대량 발생 원인**

| 원인 | 설명 |
|------|------|
| 단시간 다수 연결 | HTTP 클라이언트가 short-lived TCP 연결을 반복 생성 (Keep-Alive 미사용) |
| 리버스 프록시 | Nginx/HAProxy → 백엔드 서버로의 연결이 요청마다 새로 생성 |
| 마이크로서비스 | 서비스 간 REST 호출 시 연결 재사용 안 함 |

**해결책**

```bash
# 1. SO_REUSEADDR: 같은 포트 재사용 허용 (서버측 바인딩 시)

# 2. tcp_tw_reuse: TIME_WAIT 소켓을 outbound 연결에 재사용
sysctl -w net.ipv4.tcp_tw_reuse=1  # 클라이언트 역할 시 안전

# 3. TIME_WAIT 시간 단축 (주의: 시스템 전역 영향)
sysctl net.ipv4.tcp_fin_timeout  # FIN_WAIT2 타임아웃 (기본 60초)

# 4. Keep-Alive 활성화 (연결 재사용)
sysctl net.ipv4.tcp_keepalive_time    # 첫 keepalive 전송 (기본 7200초)
sysctl net.ipv4.tcp_keepalive_intvl   # keepalive 재전송 간격
sysctl net.ipv4.tcp_keepalive_probes  # 최대 재전송 횟수

# 현재 TIME_WAIT 수 확인
ss -s | grep TIME-WAIT
# 또는
cat /proc/net/sockstat | grep TCP
```

---

### 7.4 conntrack (Connection Tracking)

Netfilter 서브시스템의 연결 추적 모듈. NAT, stateful 방화벽의 기반.

```
패킷 도착 → conntrack이 5-tuple(src IP, src port, dst IP, dst port, proto) 해시
          → 기존 연결 테이블 조회
          → 없으면 새 엔트리 생성 (NEW 상태)
          → 있으면 기존 연결 업데이트 (ESTABLISHED/RELATED 등)
```

**conntrack 고갈 문제**
```bash
# conntrack 테이블 한도
sysctl net.netfilter.nf_conntrack_max        # 최대 엔트리 수 (기본 65536)

# 현재 사용량
cat /proc/sys/net/netfilter/nf_conntrack_count

# 고갈 시 증상: dmesg에 "nf_conntrack: table full, dropping packet"

# 해결: 한도 증가
sysctl -w net.netfilter.nf_conntrack_max=262144

# 타임아웃 단축 (오래된 연결 빨리 제거)
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600
```

**쿠버네티스와 conntrack**
- `kube-proxy`가 iptables/IPVS로 Service IP를 처리할 때 conntrack 사용
- 대규모 클러스터에서 conntrack 고갈 → Pod 간 통신 불가 장애 발생 가능

---

### 7.5 eBPF (extended Berkeley Packet Filter)

커널을 재컴파일하지 않고 커널 내부에 안전하게 프로그램을 주입하는 기술.

```
eBPF 프로그램 실행 흐름:
  C 코드 → LLVM으로 eBPF bytecode 컴파일
          → 커널 Verifier 안전성 검사 (무한 루프, 메모리 초과 등)
          → JIT 컴파일로 네이티브 코드 변환
          → Hook 포인트에 attach

Hook 포인트:
- XDP: NIC 드라이버 레벨 (가장 빠름, DDos 방어)
- TC: Traffic Control 후크
- kprobe/tracepoint: 커널 함수 진입/종료
- uprobe: 유저 스페이스 함수
- socket: 소켓 레벨 필터링
```

**주요 활용 사례**

| 도구 | 용도 |
|------|------|
| Cilium | 쿠버네티스 CNI, iptables 대체 |
| bcc / bpftrace | 커널 성능 분석, 트레이싱 |
| Falco | 보안 이상 탐지 |
| XDP | 고성능 패킷 처리, DDoS 방어 |

```bash
# bpftrace로 syscall 레이턴시 측정
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read  { @ns = hist(nsecs - @start[tid]); }'

# 실시간 TCP 이벤트 추적
tcptracer-bpfcc  # 연결/종료 이벤트
tcpretrans-bpfcc # TCP 재전송
```

---

## 8. 부팅 프로세스 & cgroup

### 8.1 부팅 단계

```
전원 ON
    ↓
[1] BIOS/UEFI
    - POST(Power-On Self Test): 하드웨어 초기화/검사
    - 부팅 가능한 디바이스 탐색 (NVMe, HDD, PXE 등)
    - MBR(Master Boot Record) 또는 GPT에서 부트로더 로드
    ↓
[2] Bootloader (GRUB2)
    - /boot/grub2/grub.cfg 로드
    - 메뉴 표시 (멀티부팅 선택)
    - 커널 이미지(vmlinuz) + initramfs(initrd) 메모리에 로드
    - 커널에 파라미터 전달 (root=, quiet, nomodeset 등)
    ↓
[3] 커널 초기화
    - 압축 해제 후 start_kernel() 실행
    - CPU, 메모리(NUMA), 인터럽트 초기화
    - 드라이버 로드
    - initramfs를 임시 루트 파일시스템으로 마운트
    - 실제 루트 파일시스템으로 전환 (pivot_root)
    ↓
[4] init / systemd (PID 1)
    - systemd: 병렬 서비스 시작 (의존성 그래프 기반)
    - 시스템 타겟 도달 (multi-user.target, graphical.target)
    ↓
[5] 서비스 기동 완료
```

**UEFI Secure Boot**
```
UEFI 펌웨어가 부트로더의 디지털 서명 검증.
미서명 부트로더/커널 모듈 → 부팅 차단.
엔터프라이즈 환경 보안 요구사항.
```

```bash
# 부팅 시간 분석
systemd-analyze blame          # 서비스별 시작 시간
systemd-analyze critical-chain # 크리티컬 패스
journalctl -b                  # 현재 부팅 로그 전체

# 커널 파라미터 확인
cat /proc/cmdline
```

---

### 8.2 cgroup v1 vs v2

**cgroup (Control Group)**: 프로세스 그룹의 자원 사용량을 제한/격리하는 커널 기능. 컨테이너 기술의 핵심.

**제어 가능한 자원**

| 자원 | 컨트롤러 |
|------|---------|
| CPU 시간 | cpu, cpuacct, cpuset |
| 메모리 | memory |
| 블록 I/O | blkio (v1) / io (v2) |
| 네트워크 | net_cls, net_prio |
| 프로세스 수 | pids |

---

**cgroup v1 구조**

```
/sys/fs/cgroup/
├── cpu/                  # CPU 제어
│   └── myapp/
│       ├── cpu.shares    # CPU 가중치 (기본 1024)
│       └── cgroup.procs  # 그룹에 속한 PID
├── memory/               # 메모리 제어
│   └── myapp/
│       ├── memory.limit_in_bytes
│       └── memory.usage_in_bytes
└── blkio/                # I/O 제어

특징: 컨트롤러별 별도 계층 구조, 복잡한 관리
```

---

**cgroup v2 구조**

```
/sys/fs/cgroup/
└── myapp/
    ├── cpu.max         # "max quota period" (예: "100000 100000" = 100% CPU)
    ├── memory.max      # 메모리 상한 (bytes 또는 "max")
    ├── memory.current  # 현재 사용량
    ├── io.max          # I/O 대역폭 제한
    └── cgroup.procs    # 속한 PID

특징: 단일 통합 계층 구조, 일관된 인터페이스, 더 강력한 격리
```

**v1 → v2 주요 변경점**

| 항목 | v1 | v2 |
|------|----|----|
| 구조 | 컨트롤러별 별도 트리 | 단일 통합 트리 |
| 위임 | 복잡 | 안전한 위임 모델 |
| 스레드 granularity | 없음 | 지원 (threaded mode) |
| writeback I/O | 미지원 | 지원 |
| 주요 사용처 | 레거시 컨테이너 | Kubernetes 1.25+, systemd |

```bash
# 현재 사용 중인 cgroup 버전 확인
stat -fc %T /sys/fs/cgroup/   # tmpfs=v1, cgroup2fs=v2

# Docker 컨테이너 cgroup 확인
docker inspect <container> | grep -i cgroup

# 프로세스가 속한 cgroup
cat /proc/<pid>/cgroup

# 메모리 제한 설정 (v2)
echo "536870912" > /sys/fs/cgroup/myapp/memory.max  # 512MB

# CPU 제한 (v2): 50% = 50000/100000
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
```

---

## 면접 포인트 요약

| 주제 | 핵심 답변 |
|------|---------|
| 가상 메모리가 왜 필요한가? | 프로세스 격리, 물리 메모리 초과 사용(Swap), COW로 fork 효율화 |
| TLB 미스가 미치는 영향 | 페이지 테이블 워크로 수백 사이클 낭비 → 컨텍스트 스위치 비용 주요 원인 |
| Mutex vs Spinlock 선택 기준 | 임계구역이 매우 짧고 선점 불가 컨텍스트면 Spinlock, 그 외는 Mutex |
| Deadlock 해결 방법 | 락 획득 순서 전역 통일 (순환 대기 조건 제거) |
| epoll이 select보다 빠른 이유 | 이벤트 발생 FD만 O(1)로 반환, 매번 전체 FD 스캔 불필요 |
| TIME_WAIT 많을 때 조치 | tcp_tw_reuse=1 + Keep-Alive 활성화 + 커넥션 풀 사용 |
| conntrack 고갈 증상 | 패킷 드롭 → 새 TCP 연결 불가, "table full" dmesg |
| OOM Killer 방지 | oom_score_adj 낮게 설정, 메모리 limit + request 적절히 설정 (k8s) |
| cgroup v2 장점 | 단일 계층 구조, 안전한 위임, writeback I/O 지원 |
| eBPF 활용 이유 | 커널 재컴파일 없이 커널 내부 로직 추가, Verifier로 안전성 보장 |

---

## 실무 진단 명령어 모음

```bash
# === 메모리 ===
free -h                          # 전체 메모리 현황
cat /proc/meminfo                # 상세 메모리 정보
vmstat 1                         # 메모리/Swap/IO 실시간
slabtop                          # Slab 캐시 사용량

# === 프로세스 ===
ps aux --sort=-%mem | head -10   # 메모리 많이 쓰는 프로세스
strace -p <pid> -c               # syscall 프로파일
pidstat -w 1                     # Context Switch 모니터링

# === 파일시스템 ===
df -ih                           # inode 사용량
lsof -p <pid>                    # 열린 파일 목록
inotifywait -m /var/log          # 파일 변경 감시

# === CPU ===
top / htop                       # CPU 사용률
mpstat -P ALL 1                  # CPU 코어별 사용률
perf top                         # CPU 핫스팟 (함수 레벨)

# === 네트워크 ===
ss -s                            # 소켓 통계 요약
ss -tnp state TIME-WAIT | wc -l  # TIME_WAIT 수
cat /proc/net/nf_conntrack | wc -l  # conntrack 사용 수
netstat -s | grep -i retrans     # TCP 재전송 통계

# === 부팅/서비스 ===
systemd-analyze blame            # 서비스 시작 시간
journalctl -b -p err             # 현재 부팅의 에러 로그
```
