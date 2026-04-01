# Linux 커널 내부 (Senior Level)

## 시스템 콜 (Syscall) 동작 원리

### User Space → Kernel Space 전환
```
User Space:
  app calls read(fd, buf, count)
  ↓
  glibc: syscall(SYS_read, fd, buf, count)
  ↓
  CPU: INT 0x80 (구버전) / SYSCALL 명령어 (x86-64)
  ↓
Kernel Space:
  privilege level 전환 (Ring 3 → Ring 0)
  레지스터 저장 (pt_regs)
  sys_call_table[syscall_nr] 로 디스패치
  sys_read() 실행
  ↓
User Space:
  결과 반환, privilege 복원 (Ring 0 → Ring 3)
```

전환 비용: ~100ns. 많은 syscall = 성능 병목.
**vDSO (Virtual Dynamic Shared Object)**: `gettimeofday`, `clock_gettime` 등 일부 syscall을 커널 진입 없이 user space에서 처리.

```bash
# 프로세스의 syscall 추적
strace -p <PID> -c           # syscall 통계 요약
strace -e trace=network ./app  # 네트워크 syscall만

# 특정 syscall 레이턴시 측정 (eBPF)
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read  { @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'
```

---

## eBPF (Extended Berkeley Packet Filter)

커널 코드를 수정하지 않고 커널 내부에서 안전하게 코드를 실행하는 기술.
커널 5.x 이상에서 full 기능 지원.

### eBPF 동작 원리
```
사용자 정의 eBPF 프로그램
↓
LLVM/Clang으로 eBPF bytecode 컴파일
↓
커널에 로드 (bpf() syscall)
↓
Verifier: 안전성 검증 (무한루프 금지, 메모리 접근 검증)
↓
JIT 컴파일 → 네이티브 머신 코드
↓
훅 포인트에 부착:
  kprobes/kretprobes (커널 함수)
  tracepoints (정적 트레이싱 포인트)
  XDP (네트워크 드라이버 레벨)
  TC (Traffic Control)
  LSM (보안 모듈)
```

### bpftrace로 프로덕션 진단

```bash
# 커널 함수 레이턴시 측정
bpftrace -e '
kprobe:vfs_read { @start[tid] = nsecs; }
kretprobe:vfs_read /@start[tid]/ {
  @read_lat = hist(nsecs - @start[tid]);
  delete(@start[tid]);
}'

# 어떤 파일이 가장 많이 읽히나
bpftrace -e '
tracepoint:syscalls:sys_enter_openat {
  @[str(args->filename)] = count();
}'

# TCP 연결 추적 (출발지/목적지)
bpftrace -e '
kprobe:tcp_connect {
  $sk = (struct sock *)arg0;
  printf("%s → %s:%d\n",
    comm,
    ntop(AF_INET, $sk->__sk_common.skc_daddr),
    $sk->__sk_common.skc_dport >> 8);
}'

# 디스크 I/O 100ms 이상 걸리는 블록 장치 요청
bpftrace -e '
tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
tracepoint:block:block_rq_complete
/@start[args->dev, args->sector]/
{
  $lat = nsecs - @start[args->dev, args->sector];
  if ($lat > 100000000) {
    printf("slow I/O: %dms, size=%d\n", $lat/1000000, args->nr_sector*512);
  }
  delete(@start[args->dev, args->sector]);
}'
```

### BCC (BPF Compiler Collection) 도구들
```bash
execsnoop    # 새로 실행되는 프로세스 추적
opensnoop    # 파일 open() 호출 추적
tcplife      # TCP 연결 수명 추적
biolatency   # 블록 I/O 레이턴시 히스토그램
runqlat      # CPU 런큐 대기 레이턴시
offcputime   # CPU off 상태 분석 (I/O, 락 대기)
profile      # CPU 프로파일링 (flame graph 생성)
```

---

## 리눅스 커널 네트워킹 스택

```
패킷 수신 경로:
NIC → DMA → Ring Buffer(RX) → NAPI polling
→ sk_buff 할당 → 이더넷 처리
→ IP 처리 (라우팅, netfilter/iptables)
→ TCP/UDP 처리
→ Socket 수신 버퍼
→ User Space (read/recv)

패킷 송신 경로:
User Space (write/send)
→ Socket 송신 버퍼
→ TCP/UDP 처리 (헤더 추가, 세그멘테이션)
→ IP 처리 (라우팅, netfilter)
→ qdisc (트래픽 제어)
→ NIC 드라이버 → 전송
```

### sk_buff (Socket Buffer)
커널 내 패킷 표현 구조체. 복사 없이 포인터 조작으로 헤더 추가/제거.

```bash
# 네트워킹 스택 지표
cat /proc/net/softnet_stat    # CPU별 패킷 처리 통계
  # 컬럼: total, dropped, time_squeeze
  # time_squeeze 증가 = NAPI budget 부족 → net.core.netdev_budget 조정

# 소켓 버퍼 현황
ss -mti                       # 소켓 메모리 사용량 상세

# 네트워크 드롭 추적
watch -d 'cat /proc/net/dev'
netstat -s | grep -i 'failed\|error\|drop'
```

### XDP (eXpress Data Path)
```
일반 경로: NIC → 커널 전체 스택 → User Space
XDP 경로:  NIC → XDP 프로그램 (드라이버 레벨, DMA 직후) → XDP_DROP/PASS/TX
```
DDoS 방어, 로드밸런싱에서 수백만 PPS 처리 가능 (Cloudflare, Facebook 사용).

---

## 메모리 서브시스템 심화

### NUMA (Non-Uniform Memory Access)
멀티소켓 서버에서 각 CPU가 자신의 메모리에 더 빠르게 접근:

```bash
numactl --hardware         # NUMA 토폴로지 확인
numastat                   # NUMA 메모리 접근 통계
  # numa_miss 증가 = 원격 NUMA 노드 접근 → 성능 저하

# 특정 NUMA 노드에서 프로세스 실행
numactl --cpunodebind=0 --membind=0 ./myapp
```

### Page Cache와 Direct I/O
```bash
# Page Cache 상태
cat /proc/meminfo | grep -E 'Cached|Buffers|Dirty|Writeback'

# Dirty Page 플러시 주기 제어
sysctl vm.dirty_ratio          # 전체 메모리 대비 dirty page 비율 (기본 20%)
sysctl vm.dirty_background_ratio  # 백그라운드 플러시 시작 비율 (기본 10%)
# SSD 서버에서 낮게 조정 (5/2) → 쓰기 지연 감소

# fsync() 빈도 높은 프로세스 확인
bpftrace -e 'kprobe:vfs_fsync { @[comm] = count(); }'
```

### Transparent Huge Pages (THP)
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# 데이터베이스(MongoDB, Redis)는 THP 비활성화 권장
# (지연 할당으로 레이턴시 스파이크 발생)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

---

## 스케줄러 (CFS - Completely Fair Scheduler)

각 프로세스에 CPU 시간을 공평하게 분배하는 레드-블랙 트리 기반 스케줄러.

```bash
# CPU 친화도 (Affinity) 설정
taskset -c 0,1 ./myapp          # CPU 0, 1에만 고정
taskset -p -c 0 <PID>           # 실행 중인 프로세스

# 실시간 우선순위 (레이턴시 중요 프로세스)
chrt -f 99 ./latency-critical   # SCHED_FIFO, 우선순위 99

# CPU 스케줄링 통계
/proc/<PID>/schedstat           # 실행 시간, 대기 시간

# 런큐 길이 모니터링
vmstat 1 | awk '{print $1}'     # r 컬럼 = 실행 대기 프로세스 수
```

### cgroup v2 CPU 제어
```bash
# CPU 가중치 설정 (기본 100, 범위 1-10000)
echo 200 > /sys/fs/cgroup/myapp/cpu.weight

# CPU 대역폭 제한 (100ms 주기 중 50ms만 사용)
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
```

---

## 실무 심화 포인트

- **Spectre/Meltdown 패치 비용**: KPTI(Kernel Page Table Isolation) 활성화 시 syscall 비용 5~30% 증가. `cat /sys/devices/system/cpu/vulnerabilities/*`로 현황 확인
- **IRQ 분산**: `cat /proc/interrupts`로 IRQ 편중 확인. `irqbalance` 또는 수동으로 `/proc/irq/<N>/smp_affinity` 설정
- **Perf 서브시스템**: `perf stat`, `perf record`, `perf top` — IPC(Instructions Per Cycle), cache miss rate, branch misprediction 측정
- **io_uring**: 최신 비동기 I/O 인터페이스. epoll 대비 syscall 오버헤드 대폭 감소. io_uring은 submission/completion 링 버퍼로 배치 처리
- **CPU Frequency Scaling**: `cpupower frequency-set -g performance` — 레이턴시 중요 서비스에서 `performance` governor 사용 (기본 `powersave`는 CPU 주파수 낮춤)
