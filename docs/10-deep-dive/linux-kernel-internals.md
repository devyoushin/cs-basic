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

## OS 업그레이드와 커널 동작

### 커널 업그레이드 전체 흐름

```
1. 패키지 설치
   apt/yum install linux-image-<version>
   ↓
2. 커널 이미지 배치
   /boot/vmlinuz-<version>       # 압축된 커널 이미지
   /boot/initrd.img-<version>    # Initial RAM Disk
   /boot/System.map-<version>    # 커널 심볼 테이블
   /lib/modules/<version>/       # 커널 모듈
   ↓
3. 부트로더(GRUB) 설정 갱신
   update-grub / grub2-mkconfig  # /boot/grub/grub.cfg 재생성
   ↓
4. 재부팅 → 새 커널로 전환
   BIOS/UEFI → GRUB → 커널 로드 → initramfs → 루트 파일시스템 마운트 → init
```

### 부팅 단계별 커널 동작

```
[1] GRUB 단계
    GRUB가 /boot/vmlinuz (커널) + /boot/initrd.img (initramfs) 를 메모리에 로드

[2] 커널 압축 해제 (Decompression)
    vmlinuz = 압축된 이미지. 커널이 스스로 압축 해제 (bzImage의 "b" = big)
    arch/x86/boot/compressed/head_64.S → decompress_kernel()

[3] 커널 초기화 (start_kernel)
    - 메모리 관리자 초기화 (mm_init)
    - 스케줄러 초기화 (sched_init)
    - IRQ, 타이머 초기화
    - 드라이버 초기화 (PCI, 블록 장치...)
    - PID 1 생성 (init/systemd)

[4] initramfs 역할
    실제 루트 파일시스템 마운트 전에 임시 루트 제공
    - 암호화 디스크 복호화 (LUKS)
    - LVM/RAID 초기화
    - 네트워크 루트 파일시스템 마운트 (iSCSI 등)
    - 필요한 커널 모듈 로드 후 실제 루트로 pivot_root

[5] systemd (PID 1)
    커널에서 제어권을 넘겨받아 사용자 공간 서비스 시작
```

### 커널 모듈 동작

커널은 모놀리식 구조지만, 드라이버 등 일부 기능을 **런타임에 로드/언로드** 가능한 모듈로 분리.

```bash
# 현재 로드된 모듈 확인
lsmod
cat /proc/modules

# 모듈 로드 / 언로드
modprobe <module>         # 의존성까지 자동 처리
modprobe -r <module>      # 언로드

# 커널 업그레이드 후 모듈 경로 갱신
depmod -a                 # /lib/modules/<version>/modules.dep 재생성

# 드라이버와 모듈 버전 확인 (ABI 불일치 문제 파악)
modinfo <module>
dkms status               # DKMS로 관리되는 서드파티 모듈 현황
```

**DKMS (Dynamic Kernel Module Support)**: 커널 업그레이드 시 서드파티 모듈(GPU 드라이버 등)을 새 커널에 맞게 자동 재컴파일. 누락 시 새 커널에서 모듈이 로드되지 않아 장애 발생.

### Live Kernel Patching (재부팅 없는 패치)

보안 패치를 적용하면서도 서비스 중단을 최소화.

```
기존 방식:
  보안 패치 배포 → 재부팅 → 서비스 다운타임 발생

라이브 패치:
  패치 모듈 로드 → 함수 포인터 교체 (ftrace 활용) → 재부팅 없이 패치 적용
```

```bash
# RHEL/CentOS: kpatch
kpatch list                    # 적용된 라이브 패치 확인
kpatch load /path/to/patch.ko  # 패치 적용

# Ubuntu: Livepatch (Canonical)
canonical-livepatch status     # 패치 현황
canonical-livepatch enable <token>

# 커널 기본 기능: livepatch 서브시스템
cat /proc/sys/kernel/ftrace_enabled   # ftrace 활성화 여부 확인
```

**동작 원리**: `ftrace`의 함수 훅을 이용해 패치 대상 함수 진입 시점에 새 함수로 리다이렉트. `klp_patch` 구조체가 old_func → new_func 매핑 관리.

### kexec: 빠른 커널 전환

BIOS/UEFI POST 단계를 건너뛰고 현재 커널에서 새 커널로 직접 점프. 재부팅 시간 1~5분 → 수 초로 단축.

```bash
# 새 커널을 메모리에 미리 로드
kexec -l /boot/vmlinuz-<new> \
      --initrd=/boot/initrd.img-<new> \
      --reuse-cmdline

# 즉시 전환 (현재 OS 종료 없이 커널 교체)
kexec -e

# systemd와 통합: systemctl kexec
systemctl kexec    # shutdown 과정 거친 후 kexec로 재부팅
```

**활용**: 고가용성이 요구되는 환경에서 커널 업그레이드 다운타임 최소화. Kubernetes 노드 드레인 후 kexec 적용.

### 커널 버전 관리와 롤백

```bash
# 설치된 커널 목록
dpkg --list | grep linux-image   # Debian/Ubuntu
rpm -qa | grep kernel            # RHEL/CentOS

# 현재 부팅된 커널
uname -r

# GRUB 기본 부팅 커널 변경 (이전 버전으로 롤백)
grub-set-default "Advanced options>Ubuntu, with Linux 5.15.0-91-generic"
update-grub

# 구버전 커널 정리 (최소 2개 유지 권장)
apt autoremove --purge           # Debian/Ubuntu (자동으로 구버전 정리)
package-cleanup --oldkernels --count=2  # RHEL

# 커널 파라미터 확인 (현재 부팅 옵션)
cat /proc/cmdline
```

### 업그레이드 시 주의 사항 & 실무 체크리스트

```
□ 업그레이드 전
  - uname -r 로 현재 커널 기록
  - lsmod > /tmp/modules-before.txt  # 로드된 모듈 목록 저장
  - dkms status  # 서드파티 모듈(GPU, 네트워크 드라이버) 확인
  - grub.cfg 백업

□ 업그레이드 후 확인
  - uname -r  # 새 커널로 부팅 확인
  - dkms status  # 모듈 재컴파일 성공 여부
  - journalctl -b -p err  # 부팅 오류 로그
  - 서비스 상태 전수 점검

□ 롤백 조건
  - 드라이버 미동작 (NIC, 스토리지 컨트롤러)
  - 커널 패닉 발생
  - 성능 회귀 (Spectre 패치 등으로 인한 syscall 비용 증가)
```

### Kernel ABI와 업그레이드 안전성

```
커널 ABI (Application Binary Interface):
  - User space ↔ kernel 간 ABI: syscall 번호·인터페이스는 리눅스가 하위 호환 보장
  - Kernel ↔ module 간 ABI: 보장하지 않음 → 서브버전 변경 시 모듈 재컴파일 필요

심볼 버전 관리:
  CONFIG_MODVERSIONS=y  # CRC로 심볼 버전 체크
  insmod 시 버전 불일치 → "disagrees about version of symbol" 오류
```

---

## Kernel Panic & kdump 분석

### Kernel Panic이란

커널이 복구 불가능한 오류를 감지했을 때 시스템을 강제 정지시키는 메커니즘.
User space crash(segfault)와 달리 커널 자체의 무결성이 깨진 상태.

```
주요 원인:
  - NULL pointer dereference in kernel code
  - Stack overflow (커널 스택은 고정 크기, 보통 8KB)
  - 잘못된 메모리 접근 (Bad Page Fault in kernel mode)
  - 하드웨어 오류 (MCE: Machine Check Exception)
  - 드라이버 버그 (커널 모듈은 커널과 같은 주소 공간에서 실행)
```

### kdump 설정 (메모리 덤프 수집)

패닉 발생 시 두 번째 커널(crash kernel)이 vmcore(메모리 덤프)를 저장.

```bash
# 1. kdump 설치
apt install kdump-tools crash   # Ubuntu
yum install kexec-tools crash   # RHEL

# 2. crash kernel용 메모리 예약 (GRUB 파라미터)
# /etc/default/grub 에 추가:
GRUB_CMDLINE_LINUX="crashkernel=256M"
update-grub && reboot

# 3. kdump 서비스 활성화
systemctl enable --now kdump

# 4. 상태 확인
kdumpctl status
cat /proc/cmdline | grep crashkernel   # 예약 메모리 확인

# vmcore 저장 위치 설정 (/etc/kdump.conf)
path /var/crash
core_collector makedumpfile -l --message-level 1 -d 31
#  -d 31 = 불필요한 페이지(zeroed, cache) 제외 → 덤프 크기 축소
```

### crash 툴로 패닉 원인 분석

```bash
# vmcore 분석 시작
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/<timestamp>/vmcore

# 주요 crash 명령어
crash> bt              # 패닉 발생 시점의 backtrace (call stack)
crash> bt -a           # 모든 CPU의 backtrace
crash> log             # 커널 로그 버퍼 (dmesg와 동일)
crash> ps             # 패닉 당시 프로세스 목록
crash> vm <PID>        # 프로세스 가상 메모리 맵
crash> files <PID>     # 열린 파일 디스크립터
crash> kmem -i         # 메모리 사용 현황

# 패닉 메시지 패턴 해석
# "BUG: kernel NULL pointer dereference at 0000000000000000"
#   → RIP(명령어 포인터)의 함수명으로 어떤 드라이버/서브시스템인지 파악
# "Call Trace:" 아래 함수 목록이 실제 오류 경로
```

### 재부팅 자동화 & 패닉 감지

```bash
# 패닉 시 자동 재부팅 설정
sysctl kernel.panic=30          # 30초 후 자동 재부팅 (0=무한대기)
sysctl kernel.panic_on_oops=1   # oops(경미한 커널 오류)도 패닉으로 처리

# /etc/sysctl.conf에 영구 적용
kernel.panic = 30
kernel.panic_on_oops = 1

# 최근 패닉 이력 확인
journalctl -b -1 -p err          # 직전 부팅의 오류 로그
ls -la /var/crash/               # 저장된 vmcore 목록
```

---

## OOM Killer 동작 원리

### OOM 발생 조건

```
물리 메모리 + Swap 모두 고갈
  → 커널이 새 페이지 할당 불가
  → OOM Killer 호출
  → oom_score가 높은 프로세스 선택 → SIGKILL
```

### oom_score 계산 방식

```bash
# 프로세스별 OOM 점수 확인 (0~1000, 높을수록 먼저 죽음)
cat /proc/<PID>/oom_score

# 점수 계산 기준:
#   메모리 사용량 비율 (RSS / 전체 물리 메모리) × 1000
#   + oom_score_adj 보정값 (-1000 ~ +1000)

# oom_score_adj: 수동 보정
cat /proc/<PID>/oom_score_adj    # 현재 값 확인

# 중요 프로세스 OOM 대상 제외 (-1000 = 완전 제외)
echo -1000 > /proc/<PID>/oom_score_adj

# systemd 서비스 단위로 설정
# /etc/systemd/system/myapp.service
[Service]
OOMScoreAdjust=-500     # 낮출수록 보호 (음수)
```

### OOM 발생 분석

```bash
# OOM 로그 확인
dmesg | grep -i 'oom\|killed process'
journalctl -k | grep -i oom

# 로그 해석 예시:
# Out of memory: Kill process 1234 (java) score 892 or sacrifice child
# Killed process 1234 (java) total-vm:8192000kB, anon-rss:7340000kB
#   → total-vm: 가상 메모리 크기, anon-rss: 실제 물리 메모리 사용량

# 메모리 사용량 상위 프로세스
ps aux --sort=-%mem | head -10
cat /proc/meminfo | grep -E 'MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree'
```

### OOM 방지 전략

```bash
# 1. Overcommit 정책 조정
sysctl vm.overcommit_memory   # 0=휴리스틱(기본), 1=항상허용, 2=제한적허용
sysctl vm.overcommit_ratio    # overcommit_memory=2일 때 물리메모리 대비 비율

# 2. cgroup v2로 메모리 상한 설정 (OOM 범위를 컨테이너로 한정)
echo "4G" > /sys/fs/cgroup/myapp/memory.max
cat /sys/fs/cgroup/myapp/memory.oom.group   # 그룹 단위 OOM 처리

# 3. 스왑 추가 (임시 완충)
fallocate -l 4G /swapfile && chmod 600 /swapfile
mkswap /swapfile && swapon /swapfile
sysctl vm.swappiness=10    # 낮을수록 swap 사용 억제 (SSD 서버 권장)
```

---

## 커널 락 메커니즘

### Spinlock

CPU가 락이 풀릴 때까지 **바쁜 대기(busy-wait)**. 컨텍스트 스위치 없음.

```
적합한 경우:
  - 락 보유 시간이 매우 짧을 때 (수십 ns)
  - 인터럽트 핸들러 내부 (sleep 불가 컨텍스트)

부적합한 경우:
  - 락 보유 중 sleep 가능성이 있을 때 → mutex 사용
  - 멀티코어에서 장시간 보유 → CPU 낭비
```

```bash
# Lock contention 진단
perf lock record -a -- sleep 5
perf lock report               # 락 경합 통계 (wait time, acquired 횟수)

# 커널 뮤텍스 경합 추적 (eBPF)
bpftrace -e '
kprobe:mutex_lock_slowpath {
  @[kstack] = count();   # 경합이 발생하는 콜스택
}'
```

### Mutex (Kernel Mutex)

락 획득 실패 시 프로세스를 **sleep** 상태로 전환. CPU 낭비 없음.

```
특징:
  - 락 소유자만 해제 가능 (소유권 있음)
  - 인터럽트 컨텍스트에서 사용 불가 (sleep 필요)
  - Priority Inheritance로 우선순위 역전 방지
```

### RCU (Read-Copy-Update)

읽기가 압도적으로 많은 데이터 구조(라우팅 테이블, 프로세스 목록 등)에 최적화된 **락 없는 읽기** 메커니즘.

```
동작 원리:
  Reader: 락 없이 읽기. rcu_read_lock() / rcu_read_unlock() 만 호출
           (preemption 비활성화만으로 충분)

  Writer: 새 데이터 구조를 복사 → 수정 → 포인터 교체
          기존 reader가 모두 빠져나간 후(Grace Period) 구버전 메모리 해제

  Grace Period: 모든 CPU가 컨텍스트 스위치를 한 번씩 거친 시점
```

```
실사용 예:
  - 커널 네트워크 라우팅 테이블 (고빈도 조회, 드문 변경)
  - 프로세스/태스크 목록 순회
  - 파일시스템 inode 캐시
```

```bash
# RCU stall 감지 (Grace Period가 너무 오래 걸릴 때)
dmesg | grep "RCU Stall"
# → CPU가 RCU read-side critical section에 너무 오래 머무름
#   원인: 무한루프, 과도한 인터럽트 비활성화

sysctl kernel.rcu_cpu_stall_timeout   # stall 감지 임계값 (기본 21초)
```

### Lock Contention 실무 진단

```bash
# perf로 커널 락 경합 측정
perf stat -e 'lock:*' -p <PID> sleep 10

# futex 경합 (user space 뮤텍스의 커널 폴백)
perf trace -e futex -p <PID> 2>&1 | head -50

# lock contention이 높은 커널 함수 찾기
bpftrace -e '
kprobe:__mutex_lock_slowpath {
  @[kstack(5)] = count();
}
interval:s:5 { print(@); clear(@); }'
```

---

## PSI (Pressure Stall Information)

커널 4.20+에서 도입. CPU/메모리/IO 자원의 **포화 정도를 정량화**.
`vmstat`/`top`의 한계(순간 스냅샷)를 극복하고 지속적인 압박을 측정.

### PSI 지표 구조

```bash
cat /proc/pressure/cpu
# some avg10=0.50 avg60=1.20 avg300=0.80 total=12345678
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

cat /proc/pressure/memory
cat /proc/pressure/io

# some: 일부 태스크가 자원 부족으로 대기한 시간 비율 (%)
# full: 모든 태스크가 대기한 시간 비율 (시스템 완전 정지 상태)
# avg10/avg60/avg300: 10초/60초/300초 이동 평균
# total: 누적 마이크로초
```

### PSI 임계값 기반 알림

```bash
# 메모리 압박이 avg10 > 10% 초과 시 알림 (cgroup 연동)
mkdir -p /sys/fs/cgroup/myapp
echo "10 500000 memory" > /sys/fs/cgroup/myapp/cgroup.pressure
#     임계값(%) 윈도우(us) 자원종류

# 실시간 PSI 모니터링
watch -n1 'cat /proc/pressure/{cpu,memory,io}'
```

### Kubernetes & PSI 연동

```bash
# kubelet의 PSI 기반 eviction (v1.21+)
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "100Mi"
# PSI full > 임계값이면 pod eviction 트리거

# 노드 PSI 현황 확인
kubectl top node
cat /proc/pressure/memory    # 노드에 직접 접근

# cAdvisor가 PSI를 컨테이너 단위로 노출
# container_pressure_cpu_some_ratio
# container_pressure_memory_some_ratio
```

### USE 방법론과 PSI 조합

```
전통적 USE:          PSI로 보완:
  Utilization         → avg300 (장기 평균 압박)
  Saturation          → some/full (대기 발생 여부)
  Errors              → dmesg, /proc/net/stat

PSI가 특히 유용한 경우:
  - 메모리 부족이 간헐적으로 발생하는 경우 (OOM 전 조기 감지)
  - 스로틀링 없는 CPU 포화 탐지
  - I/O 병목이 애플리케이션 레이턴시에 미치는 영향 정량화
```

---

## Seccomp & LSM (Linux Security Modules)

### Seccomp (Secure Computing Mode)

프로세스가 사용할 수 있는 **시스템 콜을 필터링**. 컨테이너 보안의 핵심 기반.

```
모드:
  SECCOMP_MODE_STRICT  : read/write/exit/sigreturn 만 허용 (극단적 제한)
  SECCOMP_MODE_FILTER  : BPF 규칙으로 세밀한 필터링 (Docker/Kubernetes 사용)
```

```bash
# 프로세스의 seccomp 상태 확인
cat /proc/<PID>/status | grep Seccomp
# Seccomp: 0 = 비활성, 1 = strict, 2 = filter

# Docker 기본 seccomp 프로파일 확인
docker inspect <container> | grep -i seccomp

# 컨테이너가 실제로 차단된 syscall 확인
strace -p <PID> 2>&1 | grep "Operation not permitted"
```

### 커스텀 Seccomp 프로파일 (Docker/Kubernetes)

```json
// seccomp-profile.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "exit_group"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["ptrace", "process_vm_readv"],
      "action": "SCMP_ACT_KILL"   // 위험한 syscall은 즉시 프로세스 종료
    }
  ]
}
```

```bash
# Docker에 적용
docker run --security-opt seccomp=seccomp-profile.json myapp

# Kubernetes Pod에 적용
# spec.securityContext.seccompProfile:
#   type: Localhost
#   localhostProfile: profiles/seccomp-profile.json

# 애플리케이션이 사용하는 syscall 목록 추출 (프로파일 생성 보조)
strace -qcf -e trace=all ./myapp 2>&1 | awk 'NR>2 {print $NF}' | sort -u
```

### LSM (Linux Security Modules)

커널의 보안 훅 프레임워크. AppArmor, SELinux, seccomp 모두 LSM 위에서 동작.

```
훅 삽입 위치:
  - 파일 열기/읽기/쓰기/실행
  - 프로세스 생성 (fork/exec)
  - 네트워크 소켓 연결
  - IPC 접근
  → 각 훅에서 LSM 모듈이 허용/거부 결정
```

```bash
# 활성화된 LSM 확인
cat /sys/kernel/security/lsm
# lockdown,capability,landlock,apparmor

# AppArmor (Ubuntu 기본)
aa-status                          # 프로파일 현황
aa-complain /usr/bin/myapp         # complain 모드 (위반 로그만, 차단 안 함)
aa-enforce  /usr/bin/myapp         # enforce 모드 (실제 차단)
journalctl | grep apparmor         # 차단 이벤트 확인

# SELinux (RHEL 기본)
getenforce                         # Enforcing / Permissive / Disabled
ausearch -m avc -ts recent         # 최근 SELinux 거부 이벤트
sealert -a /var/log/audit/audit.log  # 위반 분석 및 해결 제안
```

---

## Network Namespace & veth pair

컨테이너 네트워킹의 실제 커널 구현체.

### Network Namespace

프로세스 그룹마다 **독립적인 네트워크 스택** 제공 (인터페이스, 라우팅 테이블, iptables, 소켓 등).

```bash
# 네트워크 네임스페이스 생성 및 관리
ip netns add myns                  # 새 네임스페이스 생성
ip netns list                      # 목록
ip netns exec myns ip addr         # 네임스페이스 내 명령 실행
ip netns exec myns ping 8.8.8.8

# 컨테이너의 네트워크 네임스페이스 진입
PID=$(docker inspect -f '{{.State.Pid}}' <container>)
nsenter -t $PID -n ip addr         # 컨테이너 네트워크 스택 직접 확인
nsenter -t $PID -n ss -tulnp       # 컨테이너 내 소켓 목록
```

### veth pair (Virtual Ethernet)

항상 쌍으로 생성되는 가상 이더넷 인터페이스. 한쪽에 들어간 패킷이 반대쪽에서 나옴.

```
컨테이너 네트워킹 구조:

  Host Network Namespace          Container Namespace
  ┌──────────────────┐            ┌─────────────────┐
  │  docker0 (bridge)│            │   eth0          │
  │  172.17.0.1      │            │   172.17.0.2    │
  │       │          │            │       │         │
  │   veth0 (host)───┼────────────┼───veth1 (peer)  │
  └──────────────────┘            └─────────────────┘
```

```bash
# veth pair 생성
ip link add veth0 type veth peer name veth1

# veth1을 컨테이너 네임스페이스로 이동
ip link set veth1 netns myns

# 브리지에 veth0 연결
ip link set veth0 master docker0
ip link set veth0 up

# 컨테이너 측 설정
ip netns exec myns ip addr add 172.17.0.2/16 dev veth1
ip netns exec myns ip link set veth1 up
ip netns exec myns ip route add default via 172.17.0.1

# 실제 컨테이너의 veth pair 찾기
# 컨테이너 내부에서:
cat /sys/class/net/eth0/ifindex    # 인터페이스 번호 확인
# 호스트에서:
ip link | grep "^<번호>:"          # 대응하는 veth 찾기
```

### CNI (Container Network Interface)와 커널

```
Pod 생성 시 흐름:
  kubelet → CRI(containerd) → 컨테이너 생성
           → CNI 플러그인 호출 (Calico, Flannel, Cilium 등)
             → ip netns 생성
             → veth pair 생성
             → 브리지/라우팅 설정
             → iptables/eBPF 규칙 추가
```

```bash
# 클러스터 내 CNI 동작 확인
ls /etc/cni/net.d/                 # CNI 설정 파일
ls /opt/cni/bin/                   # CNI 플러그인 바이너리

# 노드의 Pod 네트워크 인터페이스 목록
ip link show | grep veth           # 각 Pod의 veth
bridge link show                   # 브리지에 연결된 인터페이스

# Cilium (eBPF 기반 CNI) 상태
cilium status
cilium endpoint list               # Pod별 eBPF 엔드포인트
```

### 네트워크 네임스페이스 성능 고려사항

```bash
# veth pair 패킷 처리 오버헤드 측정
# 컨테이너 간 통신은 veth → bridge → veth 경유
# host network 모드와 비교:
docker run --network=host myapp    # veth 오버헤드 제거 (보안 격리 포기)

# conntrack 테이블 고갈 (대규모 클러스터 주의)
sysctl net.netfilter.nf_conntrack_max          # 최대 연결 추적 수
cat /proc/sys/net/netfilter/nf_conntrack_count  # 현재 사용량
# 고갈 시 신규 연결 드롭 → 증상: "nf_conntrack: table full, dropping packet"
```

---

## 실무 심화 포인트: KPTI(Kernel Page Table Isolation) 활성화 시 syscall 비용 5~30% 증가. `cat /sys/devices/system/cpu/vulnerabilities/*`로 현황 확인
- **IRQ 분산**: `cat /proc/interrupts`로 IRQ 편중 확인. `irqbalance` 또는 수동으로 `/proc/irq/<N>/smp_affinity` 설정
- **Perf 서브시스템**: `perf stat`, `perf record`, `perf top` — IPC(Instructions Per Cycle), cache miss rate, branch misprediction 측정
- **io_uring**: 최신 비동기 I/O 인터페이스. epoll 대비 syscall 오버헤드 대폭 감소. io_uring은 submission/completion 링 버퍼로 배치 처리
- **CPU Frequency Scaling**: `cpupower frequency-set -g performance` — 레이턴시 중요 서비스에서 `performance` governor 사용 (기본 `powersave`는 CPU 주파수 낮춤)
