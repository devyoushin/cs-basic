# 모니터링 & 관찰가능성 (Observability)

## 관찰가능성의 3가지 기둥 (Three Pillars)

```
           Observability
          /      |       \
      Metrics  Logs    Traces
      (무엇이)  (왜)   (어디서)
      잘못됐나  발생했나  발생했나
```

| 기둥 | 설명 | 도구 |
|------|------|------|
| **Metrics** | 수치 데이터 (시계열) | Prometheus, CloudWatch, Datadog |
| **Logs** | 이벤트 텍스트 기록 | ELK Stack, Loki, CloudWatch Logs |
| **Traces** | 분산 요청 추적 | Jaeger, Zipkin, AWS X-Ray |

---

## 메트릭 (Metrics)

### 메트릭 타입 (Prometheus 기준)
```
Counter   : 단조 증가 (요청 수, 에러 수) → rate()로 초당 증가율 계산
Gauge     : 현재 값 (메모리, 연결 수, 온도)
Histogram : 분포 측정 → 레이턴시 p95/p99 계산에 사용
Summary   : Histogram과 유사, 클라이언트 사이드 계산
```

### Prometheus 쿼리 (PromQL)
```promql
# HTTP 초당 요청 수 (5분 평균)
rate(http_requests_total[5m])

# 에러율 (5xx / 전체)
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m])

# p99 레이턴시
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# CPU 사용률
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 핵심 대시보드 지표 (골든 시그널)
| 지표 | 설명 | 알람 기준 |
|------|------|---------|
| **Latency** | 요청 처리 시간 (p99) | SLO 초과 |
| **Traffic** | 초당 요청 수 (RPS) | 급격한 변화 |
| **Errors** | 에러율 (5xx %) | > 1% |
| **Saturation** | 리소스 포화도 | CPU > 80%, Disk > 85% |

---

## 로그 (Logs)

### 구조화 로그 (Structured Logging)
```json
{
  "timestamp": "2024-01-01T00:00:00.000Z",
  "level": "ERROR",
  "message": "Database connection failed",
  "service": "user-api",
  "trace_id": "abc-123",
  "user_id": "42",
  "duration_ms": 5000,
  "error": {
    "type": "ConnectionTimeout",
    "code": "ECONNREFUSED"
  }
}
```

텍스트 로그보다 파싱, 필터링, 집계가 훨씬 쉬움.

### 로그 레벨
```
DEBUG   → 개발 시 상세 정보 (프로덕션 비활성화)
INFO    → 정상 동작 이벤트
WARN    → 주의 필요하지만 동작 계속
ERROR   → 오류 발생, 요청 실패
FATAL   → 시스템 불능 상태
```

### ELK Stack
```
App → Filebeat → Logstash(파싱/변환) → Elasticsearch → Kibana
      (수집)      (처리)                (저장/인덱스)   (시각화)

또는 경량화:
App → Fluent Bit → Loki → Grafana
```

### 로그 분석 (Kibana/Opensearch)
```
# 최근 1시간 ERROR 로그
level: ERROR AND @timestamp: [now-1h TO now]

# 특정 서비스의 5xx
service: "payment-api" AND status_code: [500 TO 599]

# 느린 요청 (>1000ms)
duration_ms: >1000
```

---

## 분산 추적 (Distributed Tracing)

```
User Request
│
├─ API Gateway (10ms)
│   │
│   ├─ User Service (50ms)
│   │   └─ DB Query (30ms)
│   │
│   └─ Payment Service (200ms)
│       ├─ Redis (5ms)
│       └─ External API (150ms) ← 병목!
│
└─ Total: 260ms
```

### OpenTelemetry
- 관찰가능성 표준 SDK
- 메트릭 + 로그 + 트레이스 통합

```python
from opentelemetry import trace

tracer = trace.get_tracer("my-service")

with tracer.start_as_current_span("process-payment") as span:
    span.set_attribute("user.id", user_id)
    span.set_attribute("payment.amount", amount)
    result = process_payment(user_id, amount)
```

---

## 알람 설계

### 좋은 알람의 조건
1. **Actionable**: 받으면 즉시 조치 가능
2. **Relevant**: SLO에 영향을 주는 증상
3. **Noise 낮음**: 오탐(false positive) 최소화

### 알람 계층
```
P1 (Critical) → PagerDuty → 즉시 전화 (서비스 중단)
P2 (High)     → Slack #alert → 업무 시간 내 확인
P3 (Warning)  → 이메일 → 주간 검토
```

### Burn Rate Alert (에러 버짓 소진율)
```
# SLO 99.9% (에러 버짓 0.1%)
# 1시간에 에러 버짓의 2%가 소진되면 알람
burn_rate > 14.4  → 즉각 대응 (P1)
burn_rate > 6     → 확인 필요 (P2)
```

---

## Prometheus + Grafana 설정

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'my-app'
    scrape_interval: 15s
    static_configs:
      - targets: ['app:8080']
    metrics_path: /metrics

# alertmanager.yml
route:
  receiver: 'slack-notifications'
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/...'
        channel: '#alerts'
```

---

## 실무 포인트

- **메트릭 카디널리티**: 레이블 값이 너무 많으면 Prometheus 메모리 폭발. `user_id`를 레이블로 쓰지 말 것
- **로그 보존 정책**: 법적 요구사항 고려. 단기(30일) + 장기 아카이브(S3 Glacier)
- **알람 피로도 (Alert Fatigue)**: 오탐이 많으면 진짜 알람을 무시하게 됨. 정기적 알람 리뷰
- **SLO 기반 알람**: 증상(Symptom) 기반 알람. CPU 90%보다 에러율 증가가 더 중요
- **Runbook 링크**: 알람에 항상 대응 절차 문서 링크 포함
- **메트릭 4 Golden Signals**: Google SRE 책에서 제안. Latency, Traffic, Errors, Saturation
- **Continuous Profiling**: Pyroscope, Parca로 지속적 성능 프로파일링 → CPU/메모리 핫스팟 탐지
