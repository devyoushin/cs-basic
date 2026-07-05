# 시스템 신뢰성 (SRE 핵심)

## SLI / SLO / SLA

| 용어 | 정의 | 예시 |
|------|------|------|
| **SLI** (Service Level Indicator) | 서비스 수준 측정 지표 | 성공 요청 비율, p99 레이턴시 |
| **SLO** (Service Level Objective) | 내부적으로 목표하는 수준 | 99.9% 가용성, p99 < 200ms |
| **SLA** (Service Level Agreement) | 고객과의 계약 | 99.5% 가용성 보장, 위반 시 환불 |

SLO는 SLA보다 엄격하게 설정 (완충 여유 확보).

### 가용성 계산
| 가용성 | 연간 다운타임 |
|--------|-----------|
| 99% ("two nines") | 87.6시간 |
| 99.9% ("three nines") | 8.76시간 |
| 99.99% ("four nines") | 52.6분 |
| 99.999% ("five nines") | 5.26분 |

---

## 에러 버짓 (Error Budget)

```
에러 버짓 = 1 - SLO

SLO = 99.9% → 에러 버짓 = 0.1% = 월 43.2분

에러 버짓 소진율 = 실제 오류율 / (1 - SLO)
```

- 에러 버짓이 남으면 → 빠른 배포, 실험 가능
- 에러 버짓이 소진되면 → 배포 중단, 안정성 작업 우선
- **SRE의 핵심**: 에러 버짓을 통해 개발 속도와 안정성을 균형 있게 관리

---

## 고가용성 (HA) 패턴

### Active-Passive
```
Client → Load Balancer → Active Server
                      → Passive Server (대기)
         장애 발생 시 → Passive 승격
```
- Failover 시 짧은 다운타임 발생
- 리소스 낭비 (Passive가 평시 유휴)

### Active-Active
```
Client → Load Balancer → Server A (처리)
                      → Server B (동시 처리)
```
- 리소스 효율적
- 데이터 일관성 관리 복잡

---

## 장애 대응 패턴

### Circuit Breaker (서킷 브레이커)
```
CLOSED (정상) → [실패 임계치 초과] → OPEN (차단)
                                      → [일정 시간 후] → HALF-OPEN
HALF-OPEN → [성공] → CLOSED
         → [실패] → OPEN
```

- **CLOSED**: 정상 요청 처리
- **OPEN**: 즉시 오류 반환 (업스트림 보호)
- **HALF-OPEN**: 테스트 요청으로 회복 여부 확인

**라이브러리**: Resilience4j (Java), Polly (.NET), hystrix (구버전)

### Retry with Exponential Backoff
```python
import time
import random

def call_with_retry(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError as e:
            if attempt == max_retries - 1:
                raise
            # Exponential backoff + jitter
            wait = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait)
```

**Jitter(무작위 지연)**: 모든 클라이언트가 동시에 재시도하는 "Thundering Herd" 방지

### Bulkhead (격벽 패턴)
```
서비스A ─┐
서비스B ─┼→ 연결 풀 A (10개)  → 외부API
서비스C ─┘
서비스D ─→ 연결 풀 B (10개)  → 외부API
```
중요도에 따라 리소스 풀을 분리 → 하나의 서비스가 다른 서비스에 영향 못 주도록

### Timeout
- **Connection Timeout**: 연결 수립 대기 시간
- **Read/Socket Timeout**: 응답 데이터 수신 대기 시간
- **모든 외부 호출에 반드시 설정** (기본값이 없거나 너무 긴 경우 많음)

---

## 장애 복구 지표

| 지표 | 설명 |
|------|------|
| **RTO** (Recovery Time Objective) | 복구 목표 시간 (최대 허용 다운타임) |
| **RPO** (Recovery Point Objective) | 복구 목표 시점 (최대 허용 데이터 손실) |
| **MTTR** (Mean Time To Recover) | 평균 복구 시간 |
| **MTBF** (Mean Time Between Failures) | 평균 장애 간격 |

```
RTO: "장애 발생 후 1시간 내 서비스 복구"
RPO: "최근 15분 이내 데이터 복구 가능"
```

---

## 카오스 엔지니어링

- **목적**: 프로덕션에서 의도적으로 장애를 주입해 시스템 취약점 발견
- **도구**: Chaos Monkey (Netflix), Chaos Mesh (K8s), AWS Fault Injection Simulator
- **원칙**: 작게 시작, 비업무시간 실험, 자동 롤백 준비

```bash
# Chaos Mesh로 Pod 네트워크 지연 주입
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: one
  selector:
    namespaces: ["production"]
  delay:
    latency: "100ms"
    jitter: "20ms"
EOF
```

---

## 실무 포인트

- **SLO 위반 전 알람**: 에러 버짓의 2%를 1시간에 소진하면 알람 (burn rate alert)
- **포스트모텀(사후 검토)**: 장애 후 원인 분석 + 재발 방지책. 비난 없는(blameless) 문화
- **Runbook**: 장애 상황별 대응 절차 문서. 자동화된 진단 명령어 포함
- **Graceful Degradation**: 의존 서비스 장애 시 핵심 기능만 유지하고 부가 기능은 비활성화
- **Canary 배포**: 소량 트래픽에 먼저 배포 → 오류율 정상이면 전체 배포
- **Feature Flag**: 코드 배포와 기능 출시를 분리 → 위험 시 즉시 비활성화
