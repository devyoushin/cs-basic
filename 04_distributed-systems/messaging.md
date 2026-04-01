# 메시지 큐 & 이벤트 스트리밍

## 메시지 큐 vs 이벤트 스트리밍

| 구분 | 메시지 큐 | 이벤트 스트리밍 |
|------|---------|-------------|
| 소비 | 한 컨슈머가 소비 후 삭제 | 여러 컨슈머가 독립적으로 읽음 |
| 보존 | 소비 후 삭제 | 기간 설정 후 보존 |
| 순서 | 보통 FIFO | 파티션 내 순서 보장 |
| 예시 | RabbitMQ, SQS | Kafka, Kinesis |
| 사용 사례 | 작업 큐, 이메일 발송 | 이벤트 소싱, 실시간 분석 |

---

## Kafka

### 핵심 개념
```
Producer → Topic[Partition0] → Consumer Group A
                              → Consumer Group B
         → Topic[Partition1] → Consumer Group A
         → Topic[Partition2] → Consumer Group A
```

- **Topic**: 메시지 카테고리
- **Partition**: Topic의 분산 단위. 병렬 처리의 핵심
- **Consumer Group**: 같은 그룹 내에서 파티션을 나눠서 처리
- **Offset**: 파티션 내 메시지 위치. 컨슈머가 직접 관리
- **Broker**: Kafka 서버
- **Replication Factor**: 파티션 복제 수 (최소 3 권장)

### 파티션 & 순서
```
파티션 기본 할당: hash(key) % num_partitions

같은 user_id 메시지를 key로 → 동일 파티션 → 순서 보장
key 없으면 라운드로빈 → 순서 보장 안 됨
```

### 오프셋 관리
```bash
# 컨슈머 그룹 오프셋 확인
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --describe

# 오프셋 리셋 (재처리)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic my-topic \
  --reset-offsets --to-earliest --execute
```

### 전달 보장 (Delivery Semantics)
| 방식 | 설명 | 중복 위험 |
|------|------|---------|
| At-most-once | 한 번 이하 전달 (유실 가능) | 없음 |
| At-least-once | 최소 한 번 전달 (중복 가능) | 있음 |
| Exactly-once | 정확히 한 번 | 없음 (비용 높음) |

실무에서는 **At-least-once + 멱등성(Idempotency)** 조합이 일반적

---

## AWS SQS

```python
import boto3

sqs = boto3.client('sqs')

# 메시지 전송
sqs.send_message(
    QueueUrl='https://sqs.ap-northeast-2.amazonaws.com/...',
    MessageBody=json.dumps({'task': 'send_email', 'to': 'user@example.com'}),
    DelaySeconds=0
)

# 메시지 수신 및 처리
messages = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20  # Long Polling
)

for message in messages.get('Messages', []):
    process(message['Body'])
    # 처리 완료 후 반드시 삭제
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=message['ReceiptHandle']
    )
```

### SQS 핵심 설정
- **Visibility Timeout**: 메시지 처리 중 다른 컨슈머가 못 가져가는 시간 (기본 30초)
- **Dead Letter Queue (DLQ)**: 처리 실패한 메시지를 별도 큐로 이동
- **Long Polling**: 빈 응답 최소화, 비용 절감 (`WaitTimeSeconds=20`)
- **FIFO Queue**: 순서 보장 + 중복 제거 (초당 300 TPS 제한)

---

## 메시지 패턴

### Pub/Sub
```
Publisher → Message Broker → Subscriber A
                          → Subscriber B
                          → Subscriber C
```
한 메시지를 여러 구독자에게 전달.
SNS(AWS), Redis Pub/Sub, Kafka

### Work Queue (Competing Consumers)
```
Producer → Queue → Consumer A ← 병렬 처리
                → Consumer B
                → Consumer C
```
한 메시지는 하나의 컨슈머만 처리. 부하 분산.

### Request-Reply
```
Client → Queue(request) → Service
Client ← Queue(reply)   ← Service
```
비동기 RPC 패턴. 응답 큐 + Correlation ID 사용.

---

## 실무 포인트

- **Kafka 파티션 수는 나중에 늘릴 수 있지만 줄이기 어려움** → 처음부터 충분히 설정
- **Consumer Lag 모니터링**: 프로듀서와 컨슈머 처리 속도 차이. 증가하면 컨슈머 확장 필요
- **메시지 멱등성**: 같은 메시지를 두 번 처리해도 결과가 같도록 설계 (중복 처리 대비)
- **DLQ 알람 필수**: DLQ에 메시지가 쌓이면 즉시 알림 받도록 설정
- **Kafka Connect**: DB → Kafka → DB 파이프라인을 코드 없이 구성
- **Schema Registry**: Kafka 메시지 스키마 버전 관리 (Confluent Schema Registry, Avro)
- **Backpressure**: 컨슈머가 처리 못 따라갈 때 프로듀서를 늦추는 메커니즘
