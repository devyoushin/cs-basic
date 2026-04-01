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
1. 디스크(스왑)에서 해당 페이지 로드
2. 메모리 풀이면 LRU 페이지를 스왑 아웃
3. 프로세스 재개

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
