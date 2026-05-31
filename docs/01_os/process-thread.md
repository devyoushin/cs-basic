# 프로세스와 스레드

## 프로세스 vs 스레드

| 구분 | 프로세스 | 스레드 |
|------|---------|--------|
| 메모리 | 독립된 주소 공간 | 프로세스 내 메모리 공유 |
| 생성 비용 | 높음 (fork) | 낮음 |
| 통신 | IPC (파이프, 소켓, 공유메모리) | 공유 메모리 직접 접근 |
| 격리 | 강함 (하나 죽어도 다른 것 영향 없음) | 약함 (하나 크래시 → 전체 위험) |
| Context Switch | 느림 | 빠름 |

---

## PCB (Process Control Block)

OS가 프로세스를 관리하기 위한 자료구조. 프로세스별로 하나씩 존재.

```
PCB 구성 요소:
├── Process ID (PID), Parent PID (PPID)
├── 프로세스 상태 (Running/Ready/Blocked/Zombie)
├── CPU 레지스터 상태 (PC, SP, 범용 레지스터...)  ← 컨텍스트 스위칭 시 저장/복원
├── 메모리 관리 정보 (페이지 테이블 포인터, 메모리 영역)
├── 파일 디스크립터 테이블
├── 스케줄링 정보 (우선순위, CPU 사용 시간)
└── I/O 상태 (대기 중인 I/O 목록)
```

## 프로세스 상태

```
              fork()
NEW ──────→ READY ←────────────────────────┐
                │                           │
          스케줄러 선택                  I/O 완료 / 이벤트
                ↓                           │
            RUNNING ──→ WAITING (Blocked) ──┘
                │
            exit() / 비정상 종료
                ↓
             ZOMBIE ──→ 부모 wait() 시 → 소멸
```

- **Running**: CPU에서 실행 중
- **Waiting (Blocked)**: I/O 완료 대기
- **Ready**: CPU 할당 대기
- **Zombie**: 종료됐지만 부모가 exit status를 읽지 않음 (PID와 PCB만 남음)
- **Orphan**: 부모가 먼저 죽은 프로세스 (init/systemd가 입양)

```bash
# Zombie 프로세스 확인 (Z 상태)
ps aux | awk '$8=="Z"'

# Orphan/Zombie 많으면 부모 프로세스 버그 의심
# kill -9는 Zombie에 효과 없음 → 부모 프로세스 종료 필요
```

---

## CPU 스케줄링 알고리즘

OS 스케줄러가 Ready 큐에서 다음에 실행할 프로세스/스레드를 선택하는 방법.

### 주요 알고리즘

#### FCFS (First Come First Served, 선입선출)
```
도착 순서:  P1(24ms) → P2(3ms) → P3(3ms)
실행 순서:  [P1──────────────────][P2───][P3───]
대기 시간:  P1=0, P2=24, P3=27  → 평균 대기: 17ms
```
- 단순, 공평하지만 짧은 작업이 긴 작업 뒤에서 오래 기다림
- **Convoy Effect**: 긴 프로세스 하나가 전체를 지연시킴

#### SJF (Shortest Job First, 최단 작업 우선)
```
P1(6ms), P2(8ms), P3(7ms), P4(3ms)
실행 순서: [P4─][P1────][P3─────][P2──────]
```
- 평균 대기 시간 **최소** (이론적으로 최적)
- **단점**: 실행 시간을 미리 알 수 없음. 긴 작업 **기아(Starvation)** 발생 가능

#### Round Robin (시분할)
```
Time Quantum = 4ms
P1(24ms), P2(3ms), P3(3ms)

[P1──][P2─][P3─][P1──][P1──][P1──][P1──][P1──]
0    4    7   10   14   18   22   26   30
```
- 각 프로세스에 **동일한 시간(Time Quantum)** 할당 → 시간 초과 시 선점
- **공평성** 보장, 응답 시간 개선
- Time Quantum이 너무 작으면 → 컨텍스트 스위칭 오버헤드 증가
- Time Quantum이 너무 크면 → FCFS와 동일해짐
- **일반적 권장**: 10~100ms. Linux 기본 스케줄러(CFS)는 RR 기반

#### Priority Scheduling (우선순위)
```
P1(우선순위=3), P2(우선순위=1), P3(우선순위=4), P4(우선순위=2)
실행 순서: P2 → P4 → P1 → P3  (낮은 숫자 = 높은 우선순위)
```
- 우선순위 높은 프로세스 먼저 실행
- **Starvation**: 낮은 우선순위 프로세스가 무한 대기 가능
- **Aging으로 해결**: 대기 시간에 비례해 우선순위 점진 상승

#### Multilevel Queue (다단계 큐)
```
높은 우선순위 │ [시스템 프로세스]   ← 항상 먼저 실행
              │ [인터랙티브 프로세스]
              │ [배치 프로세스]
낮은 우선순위 │ [백그라운드 프로세스]
```
- 프로세스 유형별로 큐 분리, 각 큐마다 다른 알고리즘 적용
- **Multilevel Feedback Queue**: 프로세스가 행동에 따라 큐 간 이동 가능 → Linux CFS, Windows 스케줄러

### Linux CFS (Completely Fair Scheduler)
```
목표: 모든 프로세스에 공정한 CPU 시간 분배

vruntime (가상 실행 시간): 실제 실행 시간 / 가중치
→ vruntime이 가장 낮은 프로세스 선택 (Red-Black Tree)
→ nice 값으로 가중치 조절: nice -20 (높은 우선순위) ~ nice 19 (낮음)
```
```bash
# 프로세스 우선순위 조정
nice -n -10 ./high_priority_app    # 높은 우선순위로 실행
renice -n 10 -p <PID>              # 실행 중인 프로세스 우선순위 변경
```

### 스케줄링 선점 vs 비선점

| | 비선점(Non-preemptive) | 선점(Preemptive) |
|--|----------------------|-----------------|
| 개념 | 프로세스가 CPU 자발 반납 | OS가 강제로 CPU 회수 |
| 예시 | FCFS, SJF | Round Robin, Priority |
| 응답성 | 낮음 | 높음 |
| 오버헤드 | 낮음 | 컨텍스트 스위칭 비용 |
| 적합 | 배치 처리 | 인터랙티브, 실시간 |

## Context Switching

CPU가 다른 프로세스/스레드로 전환할 때:
1. 현재 프로세스 레지스터 상태(PCB)를 저장
2. 다음 프로세스 PCB를 복원
3. 실행 재개

**비용**: 캐시 무효화(Cache Miss)가 주요 오버헤드

---

## fork() vs exec()

```c
pid_t pid = fork();  // 현재 프로세스 복제

if (pid == 0) {
    // 자식 프로세스
    exec("/bin/ls", ...);  // 새 프로그램으로 교체
} else {
    // 부모 프로세스
    wait(NULL);  // 자식 종료 대기
}
```

- `fork()`: 현재 프로세스 전체 복제 (Copy-on-Write)
- `exec()`: 현재 프로세스를 새 프로그램으로 교체
- Shell이 명령어 실행할 때 fork → exec 패턴 사용

---

## IPC (Inter-Process Communication)

| 방식 | 특징 | 사용 사례 |
|------|------|---------|
| Pipe | 단방향, 부모-자식 간 | shell 파이프 (`\|`) |
| Named Pipe (FIFO) | 파일시스템 기반 | 관련 없는 프로세스 간 |
| Unix Socket | 양방향, 빠름 | nginx ↔ PHP-FPM |
| TCP Socket | 네트워크 가능 | 마이크로서비스 |
| Shared Memory | 가장 빠름 | Redis, memcached 내부 |
| Message Queue | 비동기, 버퍼링 | 작업 큐 |
| Signal | 비동기 이벤트 알림 | 종료/재로드 |

---

## 커널 스레드 vs 유저 스레드

| | 커널 스레드 | 유저 스레드 |
|--|------------|------------|
| 관리 주체 | OS 커널 | 유저 공간 라이브러리 |
| 스케줄링 | 커널 스케줄러 | 런타임 스케줄러 |
| Context Switch | 커널 모드 전환 필요 (무거움) | 유저 모드에서 처리 (가벼움) |
| 블로킹 I/O | 해당 스레드만 블로킹 | 전체 스레드 블로킹 위험 |
| 병렬 실행 | 멀티코어 활용 가능 | 단일 커널 스레드 → 진짜 병렬 불가 |
| 예시 | Linux pthread, Java Thread | Go goroutine (M:N), 과거 Green Thread |

### M:N 스레딩 (Hybrid)
- M개의 유저 스레드 → N개의 커널 스레드에 매핑
- **Go goroutine**: 수백만 개 goroutine을 수십 개 OS 스레드에 스케줄
  - 컨텍스트 스위칭 매우 빠름 (수 마이크로초)
  - Blocking syscall 시 자동으로 다른 goroutine으로 전환

### Apache MPM (Multi-Processing Module)

| MPM | 방식 | 특징 |
|-----|------|------|
| **Prefork** | 프로세스당 요청 | 안정적, 메모리 소비 큼. PHP mod_php에 필수 |
| **Worker** | 스레드당 요청 | 메모리 효율적. PHP는 thread-safe 버전 필요 |
| **Event** | 스레드 + 비동기 Keep-Alive | Worker의 개선판. Keep-Alive 연결을 별도 스레드로 처리 → 대기 연결에 스레드 낭비 없음 |

```bash
# MPM 확인
apache2 -V | grep MPM
apachectl -M | grep mpm
```

## 스레드 동기화 문제

### Race Condition
```
Thread A: x = x + 1  (read x=0, write x=1)
Thread B: x = x + 1  (read x=0, write x=1)  ← 결과: x=1 (예상: x=2)
```

### 해결 방법
- **Mutex**: 한 번에 하나의 스레드만 임계 구역 진입
- **Semaphore**: N개의 스레드 동시 접근 허용
- **Atomic Operation**: 분리 불가능한 단일 연산

### Deadlock 조건 (4가지 모두 충족 시 발생)
1. **Mutual Exclusion**: 자원을 하나의 프로세스만 사용
2. **Hold and Wait**: 자원을 가진 채로 다른 자원 대기
3. **No Preemption**: 자원을 강제로 빼앗을 수 없음
4. **Circular Wait**: A→B→C→A 순환 대기

### Thread Dump 분석 (Java)
```bash
# Thread Dump 생성
jstack <PID>
kill -3 <PID>      # SIGQUIT → 표준 출력에 Thread Dump

# 주요 패턴
# BLOCKED 상태 + "waiting to lock <0x...>" → Deadlock 후보
# WAITING on <0x...> → 락 대기 중
# RUNNABLE + 반복되는 특정 메서드 → CPU 병목
```

**Deadlock 발견 예시**:
```
"Thread-1" BLOCKED on <0x00000007> (a java.util.HashMap)
  waiting to lock owned by "Thread-2"

"Thread-2" BLOCKED on <0x00000008> (a java.util.ArrayList)
  waiting to lock owned by "Thread-1"
→ 순환 대기 → Deadlock
```

**jstack 분석 포인트**:
- `jstack <PID> | grep -A5 "BLOCKED"` → 블록된 스레드 확인
- `jstack <PID> | grep "Found.*deadlock"` → 자동 Deadlock 감지
- WAITING 스레드 다수 + 특정 락 → 락 경합 (Lock Contention)

---

## 실무 포인트

- **Nginx**: 멀티 프로세스 (Master + Worker). Worker 수 = CPU 코어 수
- **Node.js**: 단일 스레드 이벤트 루프 + libuv 스레드풀(I/O)
- **Go**: goroutine = 경량 스레드 (M:N 스케줄링)
- **컨테이너**: 프로세스 격리 (namespace + cgroup). 스레드보다 비용은 높지만 프로세스보다 가벼운 격리
- **CPU Affinity**: `taskset -c 0,1 ./app` — 특정 코어에 프로세스 고정하여 캐시 효율 향상
- **Go/Java의 GC 멈춤 (Stop-the-world)**: 짧은 레이턴시 요구 서비스에서 튜닝 필요
