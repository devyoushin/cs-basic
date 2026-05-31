# 시스템 콜 & 커널/유저 모드

## 커널 모드 vs 유저 모드

CPU는 두 가지 실행 모드를 가짐:

| | 유저 모드 (Ring 3) | 커널 모드 (Ring 0) |
|--|-------------------|-------------------|
| 접근 권한 | 제한적 (자기 메모리만) | 모든 하드웨어/메모리 |
| 실행 코드 | 애플리케이션 | OS 커널 |
| 오류 영향 | 해당 프로세스만 종료 | 시스템 크래시 (Kernel Panic) |
| 전환 방법 | 시스템 콜, 인터럽트, 예외 | iret 명령으로 복귀 |

```
Ring 0 (Kernel)  ←── 가장 높은 권한 (하드웨어 직접 접근)
Ring 1 (unused)
Ring 2 (unused)
Ring 3 (User)    ←── 애플리케이션 실행 (제한된 권한)
```

---

## 시스템 콜 (System Call)

유저 모드 프로그램이 커널 서비스를 요청하는 **유일한 인터페이스**.

### 동작 흐름

```
User Space                     Kernel Space
    │                               │
    │  write(fd, buf, n)            │
    │ ──────────────────────────→   │
    │   ① syscall 번호 설정         │
    │     (rax = 1, write)          │
    │   ② syscall 명령 실행         │
    │     (CPU 모드 전환 Ring3→0)   │
    │                           ③ 커널 핸들러 실행
    │                             (sys_write)
    │                           ④ 결과 저장 후
    │ ←────────────────────────     │
    │     ⑤ 유저 모드 복귀          │
    │       (Ring0→Ring3)           │
    │  반환값 확인                   │
```

### 모드 전환 비용
- 레지스터 저장/복원, TLB 플러시, 캐시 오염
- 컨텍스트 스위칭 비용의 주요 원인
- 시스템 콜이 많을수록 성능 저하

---

## 주요 시스템 콜

### 프로세스 관련
```
fork()      → 현재 프로세스 복제 (Copy-on-Write)
exec()      → 프로세스를 새 프로그램으로 교체
exit()      → 프로세스 종료
wait()      → 자식 프로세스 종료 대기
getpid()    → 현재 PID 반환
clone()     → 스레드 생성 (fork와 달리 공유 옵션 조절 가능)
```

### 파일/I/O 관련
```
open()      → 파일 열기 → FD 반환
read()      → FD로부터 데이터 읽기
write()     → FD에 데이터 쓰기
close()     → FD 닫기
mmap()      → 파일/메모리를 프로세스 주소 공간에 매핑
ioctl()     → 디바이스 제어 (소켓 옵션 등)
```

### 소켓 관련
```
socket()    → 소켓 생성
bind()      → 소켓에 주소 바인딩
listen()    → 연결 대기 시작
accept()    → 연결 수락
connect()   → 연결 시도
send()/recv() → 데이터 송수신
```

### 메모리 관련
```
brk()/sbrk()  → 힙 크기 조절 (malloc 내부 사용)
mmap()        → 메모리 매핑 (대용량 할당에 사용)
munmap()      → 메모리 해제
mprotect()    → 메모리 접근 권한 변경
```

---

## 시스템 콜 테이블 (Linux x86-64 일부)

| 번호 | 이름 | 설명 |
|------|------|------|
| 0 | read | 파일/소켓 읽기 |
| 1 | write | 파일/소켓 쓰기 |
| 2 | open | 파일 열기 |
| 3 | close | FD 닫기 |
| 9 | mmap | 메모리 매핑 |
| 11 | munmap | 매핑 해제 |
| 39 | getpid | PID 반환 |
| 41 | socket | 소켓 생성 |
| 42 | connect | 연결 요청 |
| 43 | accept | 연결 수락 |
| 57 | fork | 프로세스 복제 |
| 59 | execve | 프로그램 실행 |
| 60 | exit | 프로세스 종료 |
| 231 | exit_group | 스레드 그룹 전체 종료 |

```bash
# 시스템 콜 번호 조회
ausyscall --dump | head -20
```

---

## 인터럽트 & 예외

### 인터럽트 종류

| 종류 | 발생 원인 | 예시 |
|------|----------|------|
| 하드웨어 인터럽트 | 외부 장치 이벤트 | 키보드 입력, 네트워크 패킷 도착, 타이머 |
| 소프트웨어 인터럽트 | CPU 명령어 (syscall) | 시스템 콜 |
| 예외(Exception) | CPU 오류 상황 | Page Fault, Division by Zero, Segfault |

### 인터럽트 처리 흐름
```
하드웨어 이벤트 발생
    → CPU 현재 실행 중단
    → IDT(Interrupt Descriptor Table)에서 핸들러 조회
    → 커널 모드로 전환
    → 인터럽트 핸들러(ISR) 실행
    → 유저 모드로 복귀
```

---

## 컨텍스트 스위칭 비용

### 비용 구성 요소
```
① CPU 레지스터 저장 (PCB에 저장)
② 커널/유저 스택 전환
③ TLB(Translation Lookaside Buffer) 플러시
   → 이후 메모리 접근 시 Page Walk 비용 발생
④ CPU 캐시(L1/L2/L3) 오염
   → 새 프로세스가 캐시를 워밍업해야 함
```

### 성능 영향
- 컨텍스트 스위칭 자체: ~1~10 마이크로초
- 캐시 미스 복구: 수십~수백 마이크로초 추가 가능
- **잦은 syscall = 컨텍스트 스위칭 증가 = 레이턴시 증가**

### 줄이는 방법
- **io_uring**: 시스템 콜 배치 처리, 공유 링 버퍼
- **vDSO (Virtual Dynamic Shared Object)**: 일부 syscall(gettimeofday 등)을 유저 공간에서 처리
- **DPDK**: 커널 우회, 유저 공간 네트워크 스택
- **epoll**: 여러 I/O를 한 번의 syscall로 처리

---

## strace로 시스템 콜 분석

```bash
# 프로세스의 모든 시스템 콜 추적
strace ls /tmp

# 실행 중인 프로세스에 attach
strace -p <PID>

# 특정 시스템 콜만 필터링
strace -e trace=open,read,write ls /tmp
strace -e trace=network curl example.com

# 통계 출력 (어떤 syscall이 많은지)
strace -c ls /tmp
# 출력 예:
# % time     seconds  usecs/call     calls    errors syscall
# 99.90    0.000123          30         4           read
# ...

# 시간 포함 출력 (-t: 시각, -T: 소요시간)
strace -tT -p <PID>

# 자식 프로세스도 추적
strace -f bash -c "ls | grep txt"

# 특정 파일 접근 추적
strace -e trace=openat -p <PID> 2>&1 | grep "/etc"
```

### strace 활용 사례
```bash
# 앱이 어떤 설정 파일을 읽는지 확인
strace -e trace=openat myapp 2>&1 | grep "O_RDONLY"

# 네트워크 연결 문제 진단
strace -e trace=network,socket myapp 2>&1

# 느린 syscall 찾기 (1ms 이상)
strace -T -p <PID> 2>&1 | awk -F'<' '$2+0 > 0.001'

# 디스크 I/O 병목 확인
strace -e trace=read,write,fsync -T -p <PID> 2>&1
```

---

## 커널 모드 진입 경로

```
1. 시스템 콜 (syscall 명령어)
   → 프로그램이 의도적으로 커널 서비스 요청

2. 하드웨어 인터럽트
   → 네트워크 패킷 수신, 디스크 I/O 완료, 타이머

3. 소프트웨어 인터럽트 / 예외
   → Page Fault, Division by Zero, Segfault
   → SIGSEGV, SIGFPE 등의 시그널 발생
```

---

## 실무 포인트

- **`strace -c`**: 시스템 콜 통계. CPU를 많이 쓰는데 이유를 모르겠다면 먼저 실행
- **Syscall 오버헤드 측정**: `perf stat -e syscalls:sys_enter_write ./app`
- **Seccomp (Secure Computing)**: 컨테이너에서 허용할 syscall을 화이트리스트로 제한 → 공격 면적 축소. Docker 기본 Seccomp 프로파일 적용됨
- **Page Fault 급증**: `perf stat -e page-faults ./app` 또는 `vmstat 1`의 `pgflt` 항목. 메모리 부족 또는 mmap 과다 사용
- **커널 함수 추적**: `perf trace` (strace보다 오버헤드 작음), `ftrace`, `eBPF/bpftrace`
- **`/proc/<PID>/syscall`**: 현재 프로세스가 실행 중인 syscall 확인 (D 상태 프로세스 디버깅에 유용)
- **vsyscall/vDSO**: `gettimeofday()`, `clock_gettime()` 등은 커널 모드 전환 없이 처리 → 마이크로초 이하 레이턴시
