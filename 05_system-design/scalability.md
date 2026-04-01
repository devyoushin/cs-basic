# 확장성 (Scalability)

## 수직 확장 vs 수평 확장

| 구분 | 수직 확장 (Scale Up) | 수평 확장 (Scale Out) |
|------|---------------------|---------------------|
| 방법 | 서버 스펙 업그레이드 | 서버 대수 증가 |
| 한계 | 물리적 한계 존재 | 이론적 무제한 |
| 비용 | 비선형 증가 (고사양 비쌈) | 선형 증가 |
| 복잡성 | 단순 | 분산 시스템 복잡도 증가 |
| 적합 | DB, 레거시 앱 | 웹서버, 마이크로서비스 |

---

## 성능 병목 식별

### USE 방법론 (SRE 표준)
모든 리소스에 대해:
- **U**tilization: 리소스 사용률
- **S**aturation: 대기열/지연 (포화도)
- **E**rrors: 에러 수

```bash
# CPU
top / mpstat -P ALL 1
# Utilization: %us + %sy
# Saturation: load average / CPU 수

# Memory
free -h / vmstat 1
# Utilization: used / total
# Saturation: swap usage, page faults

# Disk I/O
iostat -x 1
# Utilization: %util
# Saturation: await (평균 대기 시간)

# Network
sar -n DEV 1 / nicstat
# Utilization: rxkB/s, txkB/s / max bandwidth
```

### RED 방법론 (서비스 레벨)
- **R**ate: 초당 요청 수 (RPS)
- **E**rrors: 에러율
- **D**uration: 레이턴시 (p50, p95, p99)

---

## 데이터베이스 확장

### 읽기 복제 (Read Replica)
```
Write → Primary DB
Read  → Replica 1
Read  → Replica 2
```
- 읽기 부하 분산
- 복제 지연(Replication Lag) 주의

### 샤딩 (Sharding)
```
user_id % 4 = 0 → DB Shard 0
user_id % 4 = 1 → DB Shard 1
user_id % 4 = 2 → DB Shard 2
user_id % 4 = 3 → DB Shard 3
```
- 데이터를 여러 DB에 수평 분할
- 범위 기반, 해시 기반, 디렉토리 기반
- Cross-shard 쿼리와 트랜잭션이 복잡해짐

### CQRS (Command Query Responsibility Segregation)
```
Command (쓰기) → Write Model → Event Store
Query   (읽기) → Read Model  (이벤트로 업데이트된 최적화된 뷰)
```
- 읽기/쓰기 모델 분리
- 각각 독립적으로 최적화/확장 가능

---

## 캐시 계층

```
요청 → CDN → Reverse Proxy Cache (nginx) → App Cache (Redis)
           → Application → Database Cache → Database
```

### 캐시 결정 기준
- 자주 읽고 드물게 쓰는 데이터
- 계산 비용이 높은 결과
- 외부 API 응답

---

## 비동기 처리

```
동기 (Synchronous):
Client → Server → DB → 응답 (클라이언트 대기)

비동기 (Asynchronous):
Client → Server → Queue → Worker → DB
          ↓ 즉시 응답
         (작업 ID 반환)
```

**적합한 작업**: 이메일 발송, 이미지 처리, 리포트 생성, 외부 API 연동

---

## 자동 확장 (Auto Scaling)

### 지표 기반 확장 (AWS Auto Scaling)
```yaml
# CPU 70% 이상이면 Scale Out
ScalingPolicy:
  MetricType: ASGAverageCPUUtilization
  TargetValue: 70
  ScaleOutCooldown: 60  # 확장 후 대기
  ScaleInCooldown: 300  # 축소 후 대기 (보수적)
```

### 쿠버네티스 HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## CDN (Content Delivery Network)

```
사용자 (서울) → 서울 Edge → Origin Server
사용자 (도쿄) → 도쿄 Edge (캐시 히트) → 빠른 응답
```

**캐시 대상**: 정적 파일(JS/CSS/이미지), API 응답(Cache-Control 헤더)
**무효화**: CloudFront Invalidation, Cloudflare Cache Purge

---

## 실무 포인트

- **처음부터 과도한 설계 금지**: 단순하게 시작, 병목 발생 시 확장
- **Stateless 설계**: 서버가 상태를 갖지 않으면 수평 확장이 쉬움 (세션 → Redis)
- **커넥션 풀 사이징**: `pool_size = (core_count * 2) + effective_spindle_count` (HikariCP 가이드)
- **Cold Start 대비**: Lambda, 컨테이너 등에서 처음 요청 레이턴시 높음. Warm-up 또는 최소 인스턴스 유지
- **예비 용량 (Headroom)**: 70% 사용 시 확장 트리거. 스파이크 대비 항상 여유 유지
- **병목은 항상 하나**: 최약 링크를 개선하면 다음 병목으로 이동 (Amdahl's Law)
