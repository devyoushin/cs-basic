# 메모리 관리

## 가상 메모리 (Virtual Memory)

각 프로세스는 독립된 가상 주소 공간을 가짐. 물리 메모리(RAM)와 별도로 관리.

```
프로세스 가상 주소 공간 (높은 주소 → 낮은 주소)
┌─────────────────┐ 0xFFFFFFFF
│   Kernel Space  │
├─────────────────┤ 0xC0000000
│      Stack      │ ← 아래로 증가 (지역 변수, 함수 콜)
│        ↓        │
│   (빈 공간)     │
│        ↑        │
│      Heap       │ ← 위로 증가 (동적 할당: malloc)
├─────────────────┤
│  BSS Segment    │ 초기화 안 된 전역 변수
├─────────────────┤
│  Data Segment   │ 초기화된 전역 변수
├─────────────────┤
│  Text Segment   │ 코드 (실행 가능)
└─────────────────┘ 0x00000000
```

---

## 페이징 (Paging)

가상 주소 → 물리 주소 변환 메커니즘

- **페이지**: 가상 메모리의 고정 크기 블록 (보통 4KB)
- **페이지 프레임**: 물리 메모리의 고정 크기 블록
- **페이지 테이블**: 가상→물리 주소 매핑 테이블
- **TLB (Translation Lookaside Buffer)**: 페이지 테이블 캐시

### 페이지 폴트 (Page Fault)
접근하려는 페이지가 물리 메모리에 없을 때 발생:
1. CPU가 Page Fault 예외 발생 → 커널 진입
2. 페이지 테이블 확인: 유효하지 않은 접근이면 SIGSEGV, 단순 미적재면 로드
3. 물리 메모리 여유 있으면 → 디스크(스왑)에서 해당 페이지 로드
4. 물리 메모리 부족이면 → 교체 알고리즘으로 희생 페이지 선택 → 스왑 아웃 후 로드
5. 페이지 테이블 업데이트 → 프로세스 재개

**Minor vs Major Page Fault**:
- Minor: 디스크 I/O 없음 (다른 프로세스가 이미 로드한 공유 라이브러리 등)
- Major: 디스크 I/O 필요 → 수천 배 느림

```bash
# 페이지 폴트 통계 확인
cat /proc/<PID>/stat | awk '{print "minor:", $10, "major:", $12}'
perf stat -e page-faults,major-faults ./app
```

---

## 페이지 교체 알고리즘

물리 메모리가 가득 찼을 때 어떤 페이지를 스왑 아웃할지 결정.

### FIFO (First In First Out)

```
프레임 3개, 참조 순서: 1 2 3 4 1 2 5 1 2 3 4 5

시간:   1  2  3  4  1  2  5  1  2  3  4  5
프레임: [1]          [4]          [5]          [4]
        [2]       [1][1]          [1]    [3]
           [3][4]          [2]    [2][3]    [5]
PageFault: *  *  *  *  *  *  *              *  *
총 9번 페이지 폴트
```

**Belady's Anomaly**: FIFO에서 프레임 수를 늘려도 페이지 폴트가 증가하는 역설적 현상.

### OPT (Optimal) - 이론적 최적

미래에 가장 오래 사용되지 않을 페이지 교체. **실제 구현 불가** (미래 예측 불가).
성능 비교의 기준선으로만 사용.

### LRU (Least Recently Used)

**가장 오래 전에 사용된 페이지** 교체. 참조 지역성(Locality) 원리 기반.

```
참조: 1 2 3 4 1 2 5 1 2 3

시간:  1  2  3  4  1  2  5  1  2  3
프레임:[1]          [4]    [5]    [3]
       [2]       [1]    [2]    [2]
          [3][4]          [1][1]
PageFault:* * *  *           *     *
총 6번 (FIFO보다 적음)
```

**LRU 구현 방법**:

1. **Counter 방식**: 각 페이지에 마지막 접근 시각 저장 → 가장 작은 값 교체
   - 모든 메모리 접근마다 타임스탬프 업데이트 → **오버헤드 큼**

2. **Stack 방식**: 스택으로 참조 순서 관리, 참조 시 해당 페이지를 스택 상단으로
   - 페이지 폴트 없을 때도 스택 조작 → **오버헤드 큼**

3. **Clock 알고리즘 (실제 OS 사용)**: LRU 근사치, 하드웨어 지원 활용

### Clock 알고리즘 (Second Chance)

각 페이지에 **Reference Bit (R)** 유지. 하드웨어가 페이지 접근 시 R=1 설정.

```
시계 바늘이 프레임을 순환:
  R=1인 페이지: R=0으로 초기화 후 통과 (한 번 더 기회)
  R=0인 페이지: 교체 대상 (최근에 안 쓰임)

프레임: [P1: R=1] → [P2: R=0] → [P3: R=1] → [P4: R=0]
                                               ↑ 교체 대상
바늘: P1(R=1→0, 통과) → P2(R=0, 통과) → P3(R=1→0, 통과) → P4(R=0, 교체!)
```

Linux에서 Clock 알고리즘 기반으로 Active/Inactive 리스트 관리.

### LFU (Least Frequently Used)

참조 횟수가 가장 적은 페이지 교체.

```
단점: 오래 전에 많이 쓰였지만 지금은 안 쓰는 페이지가 살아남음
     → 카운터 점진적 감소 (에이징)로 보완
```

### 알고리즘 비교

| 알고리즘 | 페이지 폴트 수 | 구현 복잡도 | Belady 이상 | 실사용 |
|---------|-------------|-----------|-----------|-------|
| FIFO | 많음 | 단순 | 발생 | X |
| OPT | 최소 | 구현 불가 | 없음 | 기준선 |
| LRU | 적음 | 복잡 | 없음 | 근사치만 |
| Clock | LRU 근사 | 보통 | 없음 | Linux, OS X |
| LFU | 보통 | 보통 | 없음 | 캐시 시스템 |

### Thrashing (스레싱)

페이지 폴트가 너무 많아 프로세스가 실질적으로 실행되지 못하는 상태.

```
메모리 부족 → 페이지 폴트 증가 → 스왑 I/O 증가
→ CPU는 스왑 대기 → CPU 사용률 낮아 보임
→ OS가 더 많은 프로세스 실행 시도 → 메모리 더 부족
→ 악순환
```

```bash
# 스레싱 징후
vmstat 1:
  si(swap in), so(swap out) 지속적으로 높음
  cs(context switch) 매우 높음
  id(idle) 거의 0인데 처리량 없음

iostat:
  디스크 I/O가 특정 장치(swap)에 집중

해결:
  - 메모리 증설
  - 프로세스 수 감소
  - Working Set 기반 스케줄링 (프로세스별 최소 프레임 보장)
```

---

## 스왑 (Swap)

- 물리 메모리 부족 시 디스크를 메모리처럼 사용
- **매우 느림** (디스크 I/O): 스왑 과다 사용 = 심각한 성능 저하
- `vmstat 1`에서 `si`(swap in), `so`(swap out) 확인
- 프로덕션 서버에서 스왑 사용량이 지속 증가하면 메모리 누수 의심

```bash
free -h           # 메모리 및 스왑 현황
swapon --show     # 스왑 장치 목록
cat /proc/meminfo | grep -i swap
```

---

## 메모리 관련 지표

```bash
# /proc/meminfo 주요 항목
MemTotal:     # 전체 물리 메모리
MemFree:      # 완전히 미사용 메모리
MemAvailable: # 실제 사용 가능 메모리 (캐시 회수 포함) ← 중요
Buffers:      # 파일시스템 메타데이터 캐시
Cached:       # 파일 내용 캐시 (page cache)
SwapCached:   # 스왑에서 불러온 후 캐시된 메모리
```

**MemAvailable** 이 실제 가용 메모리 지표 (MemFree만 보면 안 됨)

---

## OOM (Out Of Memory) Killer

메모리 고갈 시 커널이 프로세스를 선택해 강제 종료.

```bash
# OOM 발생 확인
dmesg | grep -i "out of memory"
journalctl -k | grep -i oom

# OOM Score 확인 (높을수록 종료 우선)
cat /proc/<PID>/oom_score
cat /proc/<PID>/oom_score_adj

# 중요 서비스 OOM 보호 (-1000 = 절대 종료 안 함)
echo -1000 > /proc/<PID>/oom_score_adj
```

---

## 메모리 누수 (Memory Leak) 탐지

```bash
# 프로세스 메모리 사용량 시계열 확인
ps -o pid,rss,vsz,comm -p <PID>

# valgrind (C/C++)
valgrind --leak-check=full ./myapp

# 시간에 따른 RSS 증가 모니터링
while true; do
  ps -o rss= -p <PID>
  sleep 5
done
```

---

## mmap과 Copy-on-Write

### mmap
파일이나 장치를 메모리에 직접 매핑:
```c
void *p = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
```
- 파일 I/O보다 빠름 (시스템 콜 감소)
- Redis, 데이터베이스에서 활용

### Copy-on-Write (CoW)
- `fork()` 시 즉시 메모리 복사하지 않음
- 자식이 쓰기를 시도할 때만 해당 페이지 복사
- Redis의 RDB 스냅샷이 이 방식으로 동작

---

## 실무 포인트

- **RSS vs VSZ**: RSS는 실제 물리 메모리, VSZ는 가상 메모리. RSS를 기준으로 모니터링
- **Page Cache가 메모리를 많이 사용하는 것은 정상** — Linux는 남는 메모리를 I/O 캐시로 활용
- **컨테이너 메모리 제한**: cgroup의 `memory.limit_in_bytes`. 초과 시 OOM kill 발생
- **Java Heap vs Native Memory**: JVM은 `-Xmx`로 힙 제한하지만 네이티브 메모리는 별도
- **Go 메모리**: GC가 자동 관리하지만 pprof로 메모리 프로파일링 가능
- **HugePage**: 대용량 메모리 사용 앱(DB)에서 TLB Miss 감소 → 성능 향상
