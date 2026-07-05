# 성능 엔지니어링 (Senior Level)

## Flame Graph (플레임 그래프)

CPU 시간을 어디서 소비하는지 시각화. 프로파일링의 핵심 도구.

### 생성 방법
```bash
# 1. perf로 CPU 샘플 수집 (30초간)
perf record -F 99 -p <PID> -g -- sleep 30
perf script > out.perf

# 2. Brendan Gregg의 FlameGraph 도구로 SVG 생성
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flame.svg

# 3. eBPF로 프로파일링 (perf보다 오버헤드 낮음)
profile-bpfcc -F 99 -p <PID> 30 > out.folded
./FlameGraph/flamegraph.pl out.folded > flame.svg
```

### 플레임 그래프 읽기
```
X축: 알파벳 순 정렬 (시간 아님!)
Y축: 콜 스택 깊이 (위=호출자, 아래=피호출자)
너비: CPU 시간 비율

   ┌──────────────────────────────────┐
   │         handler()  (넓을수록 핫) │
   ├────────────┬─────────────────────┤
   │ serialize()│    db_query()       │
   ├────────────┴──────┬──────────────┤
   │      request()    │   pool()     │
   └───────────────────┴──────────────┘

→ db_query가 전체의 60%를 차지하면 여기가 최적화 대상
```

### Off-CPU 플레임 그래프
CPU를 사용하지 않는 시간(I/O 대기, 락 대기) 분석:
```bash
# I/O, 락, sleep 등 대기 시간 분석
offcputime-bpfcc -p <PID> 30 > out.folded
./FlameGraph/flamegraph.pl --color=io --title="Off-CPU" out.folded > off-cpu.svg
```

---

## perf 심화

### CPU 성능 카운터
```bash
# IPC (Instructions Per Cycle): 낮으면 메모리 바운드
perf stat -e cycles,instructions,cache-misses,cache-references ./app

# 출력 해석:
#  10,000,000 cycles
#   5,000,000 instructions  #  0.5 insn per cycle  ← IPC 낮음 (정상: 2~4)
#     500,000 cache-misses  #  10.00% of all cache refs  ← 캐시 미스 높음

# LLC (Last Level Cache) 미스 - 메모리 접근 병목
perf stat -e LLC-loads,LLC-load-misses,LLC-stores ./app

# 브랜치 예측 실패
perf stat -e branch-instructions,branch-misses ./app
```

### 핫스팟 함수 찾기
```bash
# 함수별 CPU 점유율
perf top -p <PID>
perf report --stdio  # perf record 결과 분석

# 어셈블리 수준 분석 (perf annotate)
perf record -p <PID> sleep 10
perf annotate --stdio -l main  # 소스 라인별 CPU 시간
```

### 메모리 접근 패턴 분석
```bash
# NUMA 관련 메모리 접근 (Intel PMU)
perf stat -e numa-node-0,numa-node-1 ./app

# TLB 미스 (가상→물리 주소 변환 실패)
perf stat -e dTLB-loads,dTLB-load-misses ./app
# TLB 미스 높으면 HugePage 도입 고려
```

---

## 레이턴시 분석 방법론

### 레이턴시 분포 이해
```
평균은 거짓말한다.

p50:  50%의 요청이 이 시간 이하 → 일반 사용자 경험
p95:  느린 5%의 최대값
p99:  더 느린 1%
p999: 아주 가끔 발생하는 최악 케이스

예:
  p50=10ms, p99=100ms, p999=2000ms
  → 평균은 약 12ms로 보이지만 0.1%는 2초 이상!
```

### 레이턴시 스파이크 원인
```
1. GC (Garbage Collection) Stop-the-World
   → G1GC, ZGC (Java), Go GC 튜닝, pprof 분석

2. CPU 쓰로틀링 (cgroup CPU limit)
   → container_cpu_cfs_throttled_seconds_total 확인

3. OS 스케줄링 지연
   → runqlat (BCC): CPU 런큐 대기 시간 히스토그램

4. 네트워크 레이턴시 변동
   → 패킷 손실, NIC 버퍼 오버플로

5. 디스크 I/O 지연 (특히 fsync)
   → biolatency (BCC): I/O 레이턴시 분포

6. 메모리 교체 (Swapping)
   → vmstat si/so 컬럼
```

```bash
# CPU 런큐 대기 레이턴시 (스케줄링 지연)
runqlat-bpfcc -m 1  # 1초마다 ms 단위 히스토그램

# 블록 I/O 레이턴시
biolatency-bpfcc -m 1

# TCP 재전송 이벤트
tcpretrans-bpfcc
```

---

## 부하 테스트 & 벤치마킹

### 올바른 부하 테스트 방법
```bash
# wrk: HTTP 벤치마크
wrk -t12 -c400 -d30s --latency http://localhost:8080/api

# 출력:
# Latency   Avg      Stdev    Max
#   14.80ms  4.12ms  110.23ms
# Requests/sec: 25431.44

# k6: 스크립트 기반 부하 테스트
cat << 'EOF' > load.js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },  // 램프 업
    { duration: '5m', target: 100 },  // 스테이 상태
    { duration: '2m', target: 200 },  // 피크
    { duration: '2m', target: 0 },    // 쿨다운
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],  // 99%가 500ms 이하
    http_req_failed: ['rate<0.01'],     // 에러율 1% 이하
  },
};

export default function() {
  let res = http.get('http://localhost:8080/api');
  check(res, { 'status is 200': (r) => r.status === 200 });
}
EOF
k6 run load.js
```

### Coordinated Omission 문제
```
잘못된 방식: 요청을 정해진 간격으로 보내되, 응답 올 때까지 대기
→ 서버 느려지면 요청 속도 자체가 줄어들어 실제보다 좋게 측정됨

올바른 방식: 응답 기다리지 않고 스케줄에 따라 독립적으로 요청
→ wrk2, Gatling의 Open Workload Model
```

```bash
# wrk2: Coordinated Omission 해결한 버전
wrk2 -t8 -c100 -d30s -R1000 --latency http://localhost:8080/
# -R1000: 초당 1000 요청 (응답 무관하게)
```

---

## 프로파일링 도구별 비교

| 도구 | 언어 | 방식 | 적합한 경우 |
|------|------|------|-----------|
| perf | C/C++/Go/네이티브 | CPU 샘플링 | OS 레벨 포함 전체 스택 |
| async-profiler | Java/JVM | CPU + allocation | JVM 앱, 낮은 오버헤드 |
| pprof | Go | CPU/메모리/goroutine | Go 앱 기본 내장 |
| py-spy | Python | CPU 샘플링 | GIL 없이 프로파일링 |
| rbspy | Ruby | CPU 샘플링 | 프로덕션 Ruby |
| Pyroscope | 모든 언어 | 지속적 프로파일링 | 상시 성능 추적 |
| Clinic.js | Node.js | CPU + I/O | Node.js 병목 |

### Go pprof 실전
```go
import _ "net/http/pprof"

// 핸들러 등록 (자동)
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

```bash
# 30초 CPU 프로파일
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 메모리 할당 프로파일
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine 덤프 (고루틴 누수 탐지)
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# 인터랙티브 pprof
(pprof) top10        # CPU 사용 상위 10 함수
(pprof) web          # 브라우저로 flame graph
(pprof) list FuncName  # 소스 라인별 분석
```

### Java Async Profiler
```bash
# 실행 중인 JVM 프로파일링 (Signal 기반, 낮은 오버헤드)
./asprof -d 30 -f flame.html <PID>

# 메모리 할당 프로파일링
./asprof -d 30 -e alloc -f alloc.html <PID>

# Lock 경합 분석
./asprof -d 30 -e lock -f lock.html <PID>
```

---

## 시스템 전체 성능 분석 체크리스트 (60-second analysis)

```bash
# Brendan Gregg의 60-second checklist

# 1. load average
uptime

# 2. 커널 에러
dmesg | tail -20

# 3. 전체 CPU 사용률
vmstat 1 5

# 4. 프로세스별 CPU
mpstat -P ALL 1 3

# 5. 프로세스별 CPU/메모리
pidstat 1 3

# 6. 디스크 I/O
iostat -xz 1 3

# 7. 메모리
free -m

# 8. 네트워크
sar -n DEV 1 3

# 9. TCP 통계
sar -n TCP,ETCP 1 3

# 10. 프로세스 목록
top
```

---

## 실무 심화 포인트

- **Little's Law**: L = λW. 평균 요청 수(L) = 처리율(λ) × 평균 레이턴시(W). 레이턴시 증가 → 동시 요청 수 증가 → 큐 증가 → 레이턴시 더 증가 (양의 피드백 루프)
- **Amdahl's Law**: 병렬화 가능 부분이 P이면 최대 속도 향상 = 1/(1-P). 직렬 부분이 5%면 최대 20배 이상 불가
- **USE 방법론 순서**: 병목을 체계적으로 찾는 방법. 각 리소스에 대해 U→S→E 순서로 확인
- **프로덕션 프로파일링**: async-profiler, py-spy, rbspy는 실행 중인 프로세스에 연결 가능. 오버헤드 < 5%로 프로덕션에서 안전하게 사용
- **메모리 사용 패턴**: `perf mem`으로 메모리 접근 패턴 분석. False Sharing(같은 캐시 라인에 다른 스레드가 접근) 탐지
- **Tail Latency Amplification**: 마이크로서비스에서 100개 의존성이 있으면 각각 99%ile이어도 전체 성공률 = 0.99^100 = 36.6%. p99가 아닌 p99.9 모니터링 필요
