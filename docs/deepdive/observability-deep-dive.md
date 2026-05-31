# Observability 완전 분석 — eBPF, OpenTelemetry, Prometheus 내부 (Senior Level)

> 메트릭·로그·트레이스의 내부 동작부터 클라우드 네이티브 관측 아키텍처까지

---

## 1. Observability 3대 축

```
Observability = 시스템 내부 상태를 외부 출력으로 추론하는 능력

          Metrics              Logs               Traces
          ────────             ──────             ────────
목적:   "무엇이 잘못됐나?"  "왜 잘못됐나?"    "어디서 느렸나?"
형태:   숫자 시계열         텍스트 이벤트      요청 흐름 그래프
크기:   작음 (집계됨)       큼 (모든 이벤트)  중간 (샘플링)
예:     CPU 90%, p99 500ms  ERROR: null ptr    API→DB→Cache 각 레이턴시

연관성:
  Metrics → 이상 감지 → Alert
  Alert → Logs 검색 → 원인 후보
  Logs → TraceID → Trace 분석 → 정확한 지점 특정
```

---

## 2. eBPF 기반 관측

### eBPF 아키텍처

```
사용자 공간                  커널 공간
┌─────────────────┐          ┌──────────────────────────────┐
│  BPF 프로그램    │          │  eBPF 가상 머신               │
│  (C 코드)        │          │  - 64비트 RISC ISA           │
│       │          │          │  - 제한된 명령어 세트          │
│  LLVM/Clang 컴파일│          │  - 최대 100만 명령어          │
│       │          │          │  - 루프 제한 (종료 보장)       │
│  BPF bytecode   │──load──▶│  Verifier (안전성 검증)        │
│                 │          │    - NULL 포인터 검사          │
│  BPF Maps       │◀──────── │    - 메모리 범위 검사          │
│  (데이터 공유)   │          │    - JIT 컴파일 → 기계어       │
└─────────────────┘          └──────────────────────────────┘
                                      ↓ 훅 포인트들
                              kprobes / kretprobes    ← 커널 함수 진입/반환
                              tracepoints             ← 정적 계측 포인트
                              uprobe / uretprobe      ← 사용자 공간 함수
                              XDP                     ← 네트워크 드라이버 레벨
                              socket filters          ← 소켓 레벨
                              LSM hooks               ← 보안 이벤트
```

### eBPF 프로그램 예시 (CPU 레이턴시 측정)

```c
// 커널 함수 진입/반환 시간 측정
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

// 시작 시간 저장 맵 (PID → 시작 타임스탬프)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, u32);
    __type(value, u64);
} start SEC(".maps");

// 히스토그램 맵
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 64);
    __type(key, u32);
    __type(value, u64);
} dist SEC(".maps");

// 함수 진입 시 타임스탬프 기록
SEC("kprobe/vfs_read")
int kprobe_vfs_read(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 ts = bpf_ktime_get_ns();
    bpf_map_update_elem(&start, &pid, &ts, BPF_ANY);
    return 0;
}

// 함수 반환 시 레이턴시 계산
SEC("kretprobe/vfs_read")
int kretprobe_vfs_read(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *tsp = bpf_map_lookup_elem(&start, &pid);
    if (!tsp) return 0;
    u64 delta = bpf_ktime_get_ns() - *tsp;
    // 히스토그램 버킷에 기록
    u32 key = bpf_log2l(delta / 1000);  // µs 단위
    u64 *cnt = bpf_map_lookup_elem(&dist, &key);
    if (cnt) (*cnt)++;
    return 0;
}
```

### BCC / bpftrace 실무 활용

```bash
# bpftrace로 TCP 연결 레이턴시 측정
bpftrace -e '
kprobe:tcp_v4_connect { @start[tid] = nsecs; }
kretprobe:tcp_v4_connect /@start[tid]/ {
  @us = hist((nsecs - @start[tid]) / 1000);
  delete(@start[tid]);
}'

# 프로세스별 시스템 콜 횟수
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# 파일 열기 추적 (open syscall)
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# 슬로우 쿼리 탐지 (MySQL uprobe)
bpftrace -e '
uprobe:/usr/sbin/mysqld:*dispatch_command* { @start[tid] = nsecs; }
uretprobe:/usr/sbin/mysqld:*dispatch_command* /@start[tid] > 0/ {
  $ms = (nsecs - @start[tid]) / 1000000;
  if ($ms > 100) { printf("slow query: %dms pid:%d\n", $ms, pid); }
  delete(@start[tid]);
}'

# 네트워크 패킷 드롭 추적
bpftrace -e 'kprobe:kfree_skb { @[kstack] = count(); }'

# 메모리 할당 추적
bpftrace -e '
kprobe:kmalloc { @bytes = hist(arg1); }
interval:s:5 { print(@bytes); clear(@bytes); }'
```

### BCC 도구 모음

```bash
# 설치
apt-get install bpfcc-tools   # Ubuntu
yum install bcc-tools          # RHEL

# CPU 프로파일 (10초, 49Hz 샘플링)
profile-bpfcc -F 49 10 > out.stacks
flamegraph.pl out.stacks > cpu.svg

# 블록 I/O 레이턴시 히스토그램
biolatency-bpfcc

# TCP 연결 추적
tcpconnect-bpfcc
tcpaccept-bpfcc
tcpretrans-bpfcc

# 파일시스템 레이턴시
filelife-bpfcc        # 파일 수명 추적
fileslower-bpfcc 10   # 10ms 이상 느린 파일 작업
ext4slower-bpfcc 10

# 프로세스 실행 추적
execsnoop-bpfcc

# 네트워크
tcpdrop-bpfcc          # 패킷 드롭 추적
tcplife-bpfcc          # TCP 연결 수명

# 메모리
memleak-bpfcc -p <pid>  # 메모리 누수 탐지
oomkill-bpfcc           # OOM Kill 감지
```

---

## 3. Prometheus 내부 동작

### TSDB (Time Series Database) 아키텍처

```
Prometheus TSDB 구조:
  data/
    chunks_head/            ← 인메모리 Head Block (최근 2시간)
    01ABCDEF.../            ← 영구 Block (2시간씩 컴팩션)
      chunks/               ← 압축된 시계열 데이터
      index                 ← 시계열 → chunk 위치 인덱스
      meta.json             ← 블록 메타데이터
      tombstones            ← 삭제 표시

쓰기 경로:
  Scrape → WAL(Write-Ahead Log) 기록
  → 인메모리 Head Block에 추가
  → 2시간마다 Head Block → 영구 Block 컴팩션
  → 오래된 Block들 병합 (최대 31일 단위까지)

읽기 경로:
  PromQL 쿼리 → Block들 병렬 스캔
  → 시계열 인덱스로 해당 chunk 위치 찾기
  → chunk 데이터 디코딩
  → 결과 병합/집계
```

### 데이터 압축 알고리즘

```
Gorilla 압축 (Facebook):
  타임스탬프: delta-of-delta 인코딩
    첫 값: 원본
    두 번째: delta(t1 - t0)
    이후: delta-of-delta (보통 0) → 매우 높은 압축률
    15초 간격 규칙적 스크래핑: 거의 2비트로 표현

  값: XOR 기반 float64 압축
    이전 값과 XOR → 변화 없으면 0비트
    작은 변화 → 앞/뒤 0 비트 수 저장

압축률: 일반적으로 1.37 bytes/sample (원본 16bytes → 12배 압축)
```

### PromQL 실행 엔진

```
쿼리 실행 단계:
  1. 파싱 → AST(Abstract Syntax Tree) 생성
  2. 평가 시간 범위 결정 (start, end, step)
  3. 해당 범위의 시계열 선택 (인덱스 쿼리)
  4. 청크 데이터 로드 및 디코딩
  5. 함수/연산자 적용 (rate, sum, histogram_quantile 등)
  6. 결과 반환

고비용 쿼리 패턴:
  label_replace() / regex 매칭: 전체 시계열 스캔
  rate() 범위 너무 좁음: 스크래핑 간격의 4배 이상 권장
  high cardinality labels: 수백만 시계열 → 메모리/CPU 폭발

Cardinality 문제:
  label에 user_id, session_id, request_id 등 넣으면 안 됨
  허용: status_code, method, region (카디널리티 낮음)
  금지: uuid, ip_address (카디널리티 무한)
```

```bash
# Prometheus 내부 상태 API
curl localhost:9090/api/v1/status/tsdb | jq .
# headStats.numSeries: 현재 시계열 수
# headStats.chunkCount: 청크 수

# 카디널리티 높은 메트릭 확인
curl localhost:9090/api/v1/status/tsdb?limit=20 | \
  jq '.data.seriesCountByMetricName[] | select(.count > 10000)'

# WAL 크기 확인
du -sh /prometheus/wal/

# 쿼리 성능 분석 (느린 쿼리)
curl 'localhost:9090/api/v1/query_range?query=slow_query&...' \
  --trace-time
```

### Prometheus 확장: Remote Write / Remote Read

```
Thanos / Cortex / VictoriaMetrics:
  장기 보존: Prometheus 기본 최대 수개월 → 오브젝트 스토리지 (S3)에 장기 저장
  고가용성: HA Prometheus 쌍의 중복 제거
  글로벌 쿼리: 여러 Prometheus의 통합 쿼리

Remote Write 흐름:
  Prometheus → HTTP POST → 원격 저장소
  WAL tail → 배치 인코딩 → protobuf+snappy 압축 → 전송
  실패 시 WAL에서 재전송 (최대 2시간 버퍼)

VictoriaMetrics 장점:
  Prometheus보다 3~5× 낮은 메모리/디스크
  자체 압축 알고리즘 (더 높은 압축률)
  초고카디널리티 지원
```

---

## 4. OpenTelemetry 내부

### OTel 아키텍처

```
앱 (SDK)                    Collector                  백엔드
┌─────────────┐             ┌──────────────────┐       ┌────────────┐
│ OTel SDK    │             │ Receiver         │       │ Jaeger     │
│  Tracer     │──OTLP/gRPC─▶│  OTLP, Jaeger   │──────▶│ Zipkin     │
│  Meter      │             │  Prometheus      │       │ Tempo      │
│  Logger     │             │                  │       ├────────────┤
│  Propagator │             │ Processor        │       │ Prometheus │
│  Exporter   │             │  Batch, Filter   │       │ Cortex     │
└─────────────┘             │  Tail Sampling   │       ├────────────┤
                            │  Attributes      │       │ Loki       │
                            │                  │       │ Elasticsearch│
                            │ Exporter         │       └────────────┘
                            │  OTLP, Jaeger... │
                            └──────────────────┘
```

### 트레이싱 내부 동작

```
Span 생성:
  tracer.Start(ctx, "operation-name")
  → SpanContext 생성:
      TraceID: 16바이트 랜덤 (요청 전체에 고유)
      SpanID: 8바이트 랜덤 (각 작업에 고유)
      TraceFlags: 01=샘플링됨, 00=샘플링 안 됨
      TraceState: 벤더 별 추가 상태

Context Propagation:
  HTTP: W3C Trace Context 헤더
    traceparent: 00-<traceId>-<spanId>-<flags>
    tracestate: vendor=specific

  gRPC: metadata로 전달

서비스 A → 서비스 B:
  A가 outgoing request에 traceparent 헤더 삽입
  B가 수신 시 헤더 추출 → 부모 SpanID로 자식 Span 생성
  → 전체 호출 트리 구성
```

### 샘플링 전략 비교

```
Head Sampling (요청 시작 시 결정):
  장점: 구현 단순, 오버헤드 낮음
  단점: 에러/느린 요청을 놓칠 수 있음

  AlwaysOn: 100% 기록 (개발 환경)
  AlwayOff: 0% 기록
  TraceIdRatio: TraceID 해시 기반 일정 비율
    1%: 트래픽 많은 서비스에 적합

Tail Sampling (요청 완료 후 결정):
  장점: 에러/느린 요청 100% 포착
  단점: 모든 Span을 버퍼에 유지해야 함 → 메모리 비용

  OTel Collector Tail Sampling 정책:
    - 에러 Span 포함 시 100% 보존
    - 레이턴시 > 500ms 100% 보존
    - 나머지 1% 랜덤 샘플링

Collector 설정:
  processors:
    tail_sampling:
      decision_wait: 10s
      policies:
        - name: errors-policy
          type: status_code
          status_code: {status_codes: [ERROR]}
        - name: slow-traces-policy
          type: latency
          latency: {threshold_ms: 500}
        - name: probabilistic-policy
          type: probabilistic
          probabilistic: {sampling_percentage: 1}
```

### OTLP Exporter 배치 처리

```
SDK 내부 배치 처리:
  Span이 종료되면 SpanProcessor로 전달
  BatchSpanProcessor:
    - maxQueueSize: 2048 (기본)
    - scheduledDelayMillis: 5000ms (5초마다 플러시)
    - maxExportBatchSize: 512
    → 512개 Span 또는 5초마다 Exporter로 전송

  SimpleSpanProcessor (동기식):
    - Span 종료 즉시 Export → 레이턴시 영향
    - 개발/디버깅 용도

Exporter 연결 실패 시:
  재시도 (exponential backoff)
  큐가 가득 차면 → 새 Span 드롭 (손실 발생)
  → maxQueueSize, scheduledDelayMillis 튜닝 필요
```

---

## 5. 로그 파이프라인 내부

### 구조화 로그와 로그 레벨

```
비구조화 vs 구조화:
  비구조화: "2024-01-01 ERROR user 12345 login failed"
  구조화 (JSON): {"time":"2024-01-01","level":"ERROR","event":"login_failed","user_id":12345}

구조화 로그의 이점:
  → 필드 기반 쿼리 가능 (Loki, Elasticsearch)
  → 집계 용이 (user_id별 에러 수 등)
  → 카디널리티 통제 가능

로그 레벨 활용:
  DEBUG: 개발 환경만 (운영에서 비활성화)
  INFO: 주요 상태 변화 (요청 처리, 서비스 시작)
  WARN: 비정상이지만 서비스 계속 가능
  ERROR: 처리 실패, 즉시 대응 필요
  FATAL/CRITICAL: 서비스 종료 레벨
```

### Loki 아키텍처

```
Loki 핵심 설계: "Prometheus, but for logs"
  메트릭처럼 Label 기반 인덱싱
  로그 내용은 인덱싱 안 함 (Elasticsearch와 차이)

컴포넌트:
  Distributor: 수신, 검증, 레이블 해싱
  Ingester: 인메모리 청크 버퍼, 주기적 플러시
  Querier: 쿼리 처리, 오브젝트 스토리지 조회
  Query Frontend: 쿼리 샤딩, 캐싱
  Compactor: 오브젝트 스토리지 청크 압축/인덱스 정리
  Ruler: LogQL 기반 알림 규칙

스토리지:
  인덱스: DynamoDB, Cassandra, BoltDB, TSDB(기본)
  청크: S3, GCS, Azure Blob, 로컬 파일시스템

로그 청크:
  동일 레이블 세트의 로그를 시간순 청크로 묶음
  snappy 압축 → 높은 압축률 (로그는 반복적)
  타임스탬프 오름차순 강제 (오래된 로그 거부)
```

### LogQL 쿼리

```logql
# 기본 로그 스트림 선택
{app="nginx", env="production"}

# 필터링
{app="nginx"} |= "error"                    # 포함
{app="nginx"} != "health"                   # 제외
{app="nginx"} |~ "status=[45][0-9]{2}"      # 정규식

# 파싱 (구조화)
{app="api"} | json                          # JSON 파싱
{app="nginx"} | pattern `<ip> - - [<time>] "<method> <path> HTTP/<ver>" <status> <bytes>`

# 메트릭 쿼리 (LogQL v2)
rate({app="api"} |= "error" [5m])           # 에러 발생률
count_over_time({app="api"}[1h])            # 시간당 로그 수

# 집계
sum by (status) (rate({app="nginx"} | json | __error__="" [5m]))

# p99 레이턴시 (로그에서 추출)
quantile_over_time(0.99,
  {app="api"} | json | unwrap duration_ms [5m]) by (endpoint)
```

---

## 6. 분산 트레이싱 분석 패턴

### 레이턴시 기여도 분석

```
Trace 예시:
  [전체: 450ms]
  ├── API Handler [450ms]
  │   ├── Auth Check [5ms]
  │   ├── DB Query [300ms]  ← 병목
  │   │   ├── Connection Pool Wait [150ms]  ← 풀 고갈
  │   │   └── Query Execute [150ms]
  │   ├── Cache Check [10ms]
  │   └── Response Serialize [135ms]  ← 2순위

분석:
  Critical Path: 가장 긴 순차 경로 탐색
  Concurrent Span: 병렬 실행 여부 확인
  Gap: 부모-자식 Span 사이 공백 → 계측되지 않은 코드
```

### Exemplar — 메트릭과 트레이싱 연결

```
Prometheus Exemplar:
  히스토그램/서머리 메트릭에 TraceID를 샘플로 첨부
  → 메트릭 이상 → 해당 시점 TraceID → 트레이스로 직접 이동

예:
  http_request_duration_seconds_bucket{le="0.5"} 1234 # {traceID="abc123"} 0.48

Grafana 활용:
  메트릭 패널 → Exemplar 점 클릭 → Tempo 트레이스 자동 연동
  → Metrics to Traces 자동 연결

설정 (Prometheus):
  --enable-feature=exemplar-storage
  앱 측: OpenMetrics 형식으로 Exemplar 포함하여 노출
```

---

## 7. SLI/SLO 실무 구현

### SLO 기반 Alerting

```
에러 버짓 소진율 기반 알림:
  SLO: 30일 99.9% (허용 에러 버짓: 43.2분)

  Critical: 1시간 에러율 × 720 > 에러버짓
    → 1시간 창에서 버짓 전체 소진 속도
    → 즉시 대응

  Warning: 6시간 에러율 × 120 > 에러버짓
    → 6시간 창에서 소진 속도
    → 업무 시간 대응

PromQL 예시:
  # 1시간 소진율 (burn rate > 14.4배 → 즉시 알림)
  (
    rate(http_requests_total{status=~"5.."}[1h])
    /
    rate(http_requests_total[1h])
  ) > 0.001 * 14.4   # 14.4 = 30일 버짓 / 1시간

  # 멀티 윈도우 번 레이트 (구글 SRE 권장)
  (
    (
      rate(errors[1h]) / rate(requests[1h]) > bool 14.4 * 0.001
    ) and (
      rate(errors[5m]) / rate(requests[5m]) > bool 14.4 * 0.001
    )
  )
```

### RED / USE / Four Golden Signals

```
RED (마이크로서비스):
  Rate:   요청 수 (requests/s)
  Error:  에러율 (errors/s 또는 %)
  Duration: 레이턴시 분포 (p50, p95, p99)

USE (인프라 리소스):
  Utilization: 리소스 사용률 (CPU 70%)
  Saturation:  포화도 (큐 깊이, 대기 중인 작업)
  Errors:      에러 발생 수

Four Golden Signals (Google SRE):
  Latency:    응답 시간 (성공/실패 분리)
  Traffic:    요청 부하 (RPS, 연결 수)
  Errors:     에러율
  Saturation: 리소스 포화 (예: CPU 포화 = throttling 발생)

우선순위:
  일반: RED (서비스 관점)
  인프라: USE (노드/컨테이너 관점)
  통합: Four Golden Signals
```

---

## 8. 실무 진단 도구 모음

### 메트릭 수집 상태 확인

```bash
# Prometheus 타겟 상태
curl localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job, health, lastError}'

# 스크래핑 실패 메트릭
up{job="my-service"} == 0

# 메트릭 카디널리티 확인
curl localhost:9090/api/v1/label/__name__/values | jq '.data | length'

# 시계열 수 상위 메트릭
curl 'localhost:9090/api/v1/status/tsdb?limit=20' | \
  jq '.data.seriesCountByMetricName | sort_by(.count) | reverse | .[0:10]'

# OTel Collector 파이프라인 메트릭
curl localhost:8888/metrics | grep otelcol_processor
# otelcol_processor_dropped_spans: 드롭된 Span 수
# otelcol_exporter_queue_size: 익스포터 큐 크기
```

### 로그 레이턴시 분석 (Loki + LogQL)

```logql
# 최근 1시간 에러 수 추이
sum(count_over_time({app="api"} |= "ERROR" [5m]))

# 슬로우 쿼리 감지 (JSON 로그에서 duration_ms 추출)
{app="api"} | json | duration_ms > 1000 | line_format "{{.timestamp}} {{.path}} {{.duration_ms}}ms"

# 스택 트레이스 멀티라인
{app="java-app"} |= "Exception" | multiline(firstline=`^\d{4}-\d{2}-\d{2}`)
```

### Jaeger / Tempo 트레이스 분석

```bash
# Jaeger API로 느린 트레이스 조회
curl "localhost:16686/api/traces?service=my-service&minDuration=1000ms&limit=20"

# TraceID로 특정 트레이스 조회
curl "localhost:16686/api/traces/<traceId>"

# Tempo (Grafana) HTTP API
curl "localhost:3200/api/traces/<traceId>"
```

---

## 9. 실무 심화 포인트 & 면접 대비

### "eBPF로 무엇을 할 수 있는가?"
```
관측:
  애플리케이션 수정 없이 레이턴시, 오류, 리소스 사용량 측정
  시스템 콜 추적, 네트워크 흐름 분석

보안:
  비정상 프로세스 실행 감지 (falco, Cilium Tetragon)
  네트워크 정책 시행

네트워크:
  XDP: 와이어스피드 패킷 처리 (DDoS 방어, LB)
  Cilium: iptables 없이 순수 eBPF로 쿠버네티스 네트워크

제약:
  커널 4.9+ 필요 (기능별로 최소 버전 다름)
  Verifier가 허용하는 프로그램만 실행
  루프, 무한 실행 불가
```

### "Prometheus 카디널리티 문제 해결 방법?"
```
원인:
  레이블 값에 무한한 값 사용 (user_id, request_id, IP)

탐지:
  TSDB status API로 상위 메트릭 확인
  recording_rules로 집계 후 원본 드롭

해결:
  1. 고카디널리티 레이블 제거
  2. 집계 Recording Rule로 미리 집계
  3. 메트릭 릴레이블링으로 레이블 드롭
     metric_relabel_configs:
       - regex: user_id
         action: labeldrop
  4. VictoriaMetrics 등 고카디널리티 지원 스토리지 도입
```

### "Tail Sampling 구현 시 주의점?"
```
모든 Span이 Collector에 도달해야 함:
  → Collector 단일 인스턴스 또는 일관된 해싱 필요
  → TraceID 기준으로 같은 Collector 인스턴스로 라우팅

메모리 비용:
  decision_wait 동안 Span을 버퍼에 유지
  트래픽 많으면 → 메모리 폭발

권장 아키텍처:
  앱 → Load Balancer (TraceID 기준 해시) → Collector Pool → 백엔드
  Collector Pool: 수평 확장. 각 인스턴스가 특정 TraceID 담당
```

| 질문 | 핵심 답변 |
|------|-----------|
| Metrics vs Logs vs Traces | 메트릭=무엇, 로그=왜, 트레이스=어디서 |
| eBPF vs 기존 관측 방식 | 앱 수정 없이 커널 레벨 계측. 오버헤드 최소 |
| Prometheus 데이터 모델 | (메트릭명 + 레이블 집합) = 하나의 시계열 |
| OTLP 장점 | 표준 프로토콜. 벤더 독립. SDK와 백엔드 분리 |
| 에러 버짓 번 레이트 | SLO 위반 속도 정량화. 빠를수록 즉각 대응 |
| Exemplar 목적 | 메트릭 이상 → 해당 시점 TraceID → 직접 이동 |
