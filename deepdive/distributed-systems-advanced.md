# 분산 시스템 심화 (Senior Level)

## Raft 합의 알고리즘 내부

### Leader Election
```
모든 노드는 Follower로 시작.
Election Timeout (150~300ms 랜덤) 동안 heartbeat 없으면:
  1. Follower → Candidate
  2. 자신의 term 증가
  3. 자신에게 투표 후 RequestVote RPC 브로드캐스트

투표 규칙:
  - 이번 term에 한 표만
  - 후보의 로그가 자신보다 최신이어야 수락
  → 과반수(N/2 + 1) 확보 → Leader

Split Vote 발생 시:
  - 타임아웃 → 새 term으로 재선거
  - 랜덤 타임아웃으로 split vote 반복 확률 낮춤
```

### Log Replication
```
Client → Leader (AppendEntries RPC)
           ↓ 동시에 모든 Follower에 전송
         Follower 1: 수신 확인
         Follower 2: 수신 확인
         Follower 3: (느린 노드, 타임아웃)
           ↓ 과반수(2/3) 확인
         Leader: 로그 커밋 + 클라이언트에 응답
           ↓ 다음 heartbeat에
         Follower 3: 따라잡기 (catching up)
```

```bash
# etcd Raft 상태 확인
etcdctl endpoint status --cluster -w table

# 리더 확인
etcdctl endpoint status | grep true

# Raft 제안 처리 현황
etcdctl endpoint status -w json | jq '.[].Status.raftAppliedIndex'

# etcd 성능 벤치마크
etcdctl check perf --load="s"  # small, medium, large

# etcd 느린 이유 분석 (디스크 I/O가 주 원인)
etcdctl endpoint status -w json | jq '.[].Status.dbSize'
# WAL fsync 지연이 99ms 초과하면 etcd 로그에 경고
```

### 선형화 가능성 (Linearizability)
```
요청 시작 ~ 응답 완료 사이 어떤 시점에 원자적으로 실행됐다고 볼 수 있어야 함.

예:
T1: Write(x=1) [----]
T2:    Read(x)    [--] → 반드시 1 반환 (T1이 완전히 포함)

Raft는 선형화 가능한 읽기를 위해 Leader가 quorum 확인 필요
(Lease Read: 리더 임기 중 시계 기반으로 quorum 없이 읽기 가능 — 클락 드리프트 주의)
```

---

## Consistent Hashing

### 문제: 단순 모듈러 해싱의 한계
```
서버 3대: hash(key) % 3
서버 추가/제거 시: 대부분의 키 재배치 → 캐시 무효화 대폭 발생
```

### Consistent Hashing 동작
```
해시 링 (0 ~ 2^32):

         0
    Server A
  /          \
Server C    Server B
  \          /
   2^32/2

키 → 해시값 → 시계방향으로 첫 서버에 배치

서버 추가/제거 시: 인접한 구간의 키만 이동
```

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes=None, replicas=150):
        self.replicas = replicas  # 가상 노드 수 (균등 분산)
        self.ring = {}
        self.sorted_keys = []

        for node in (nodes or []):
            self.add_node(node)

    def add_node(self, node):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)

    def remove_node(self, node):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            del self.ring[key]
            self.sorted_keys.remove(key)

    def get_node(self, key):
        if not self.ring:
            return None
        hash_key = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
```

**가상 노드(Virtual Node/Replica)**: 서버를 여러 위치에 배치하여 균등 분산.
Cassandra, Redis Cluster, Memcached(ketama)에서 사용.

---

## 분산 트랜잭션 심화

### Percolator (Google BigTable 기반 분산 트랜잭션)
TiDB가 채택한 낙관적 분산 트랜잭션 프로토콜:
```
1. Prewrite (락 + 데이터 쓰기):
   - Primary Key에 lock 기록
   - 모든 Secondary Key에 lock 기록 (primary 참조)

2. Commit:
   - Timestamp Oracle(TSO)에서 commit_ts 발급
   - Primary Key의 lock을 commit 레코드로 교체 (원자적 커밋)
   - Secondary Key의 lock 비동기 정리

3. 충돌 시:
   - 다른 트랜잭션이 lock 발견 → primary 상태 확인
   - primary가 committed → secondary를 직접 커밋
   - primary가 아직 미커밋 → 만료 시 롤백
```

### Saga 패턴 구현 심화
```python
# Choreography Saga: 이벤트 기반
class OrderSaga:
    def handle_order_created(self, event):
        # Payment Service가 이벤트 구독
        try:
            self.payment_service.charge(event.order_id, event.amount)
            self.emit('payment.completed', event.order_id)
        except PaymentFailed:
            self.emit('payment.failed', event.order_id)  # 보상 이벤트

    def handle_payment_failed(self, event):
        # Order Service가 구독 → 주문 취소
        self.order_service.cancel(event.order_id)

# Orchestration Saga: 중앙 조율자
class OrderOrchestrator:
    def execute(self, order_id):
        steps = [
            Step(self.reserve_inventory, self.release_inventory),
            Step(self.process_payment, self.refund_payment),
            Step(self.create_shipment, self.cancel_shipment),
        ]
        completed = []
        try:
            for step in steps:
                step.execute(order_id)
                completed.append(step)
        except Exception:
            for step in reversed(completed):
                step.compensate(order_id)  # 역순으로 보상
            raise
```

---

## 벡터 클락 & 인과 일관성

### 벡터 클락
```
각 노드가 [N1: v1, N2: v2, N3: v3] 형태의 클락 유지

N1이 이벤트 발생: N1의 카운터 +1 → [1, 0, 0]
N1이 N2에 메시지 전송: 자신의 클락 포함
N2가 수신: max(N2의 클락, 수신 클락) + N2 카운터 +1 → [1, 1, 0]

비교:
A happens-before B: A의 모든 카운터 ≤ B의 대응 카운터 (최소 하나 <)
동시 이벤트: 어느 쪽도 happens-before 아님 → 충돌 해결 필요
```

### CRDT (Conflict-free Replicated Data Types)
충돌 없이 병합 가능한 데이터 구조:
```python
# G-Counter (증가만 가능한 카운터)
class GCounter:
    def __init__(self, node_id, num_nodes):
        self.node_id = node_id
        self.vector = [0] * num_nodes

    def increment(self):
        self.vector[self.node_id] += 1

    def value(self):
        return sum(self.vector)

    def merge(self, other):
        # 각 노드의 최대값으로 병합 (항상 수렴)
        self.vector = [max(a, b) for a, b in zip(self.vector, other.vector)]

# 활용: Redis의 HyperLogLog, Riak의 기본 데이터 타입
# 결과적 일관성 시스템에서 충돌 없는 분산 카운터
```

---

## 분산 시스템 시간 문제

### NTP의 한계
```
물리적 시계는 드리프트됨 (약 ±100ms/day).
NTP로 동기화해도 수십ms 오차 가능.
→ "같은 시간"이라도 노드마다 다른 값을 관찰할 수 있음.
```

### Google Spanner의 TrueTime
```
GPS + 원자시계로 불확실성 구간 제공:
  TT.now() → [earliest, latest] (약 ±7ms)

커밋 시:
  1. TT.now()로 commit_ts 선택
  2. TT.after(commit_ts) 될 때까지 대기 (commit wait)
  3. 커밋 완료
→ 외부 일관성 보장: 트랜잭션 A가 B보다 먼저면 ts(A) < ts(B) 항상 성립
```

### HLC (Hybrid Logical Clock)
```python
import time

class HLC:
    """Hybrid Logical Clock: NTP 시계 + 논리 카운터"""
    def __init__(self):
        self.l = 0  # 최대 물리 시간 (ms)
        self.c = 0  # 카운터

    def now(self):
        pt = int(time.time() * 1000)  # 현재 물리 시간
        if pt > self.l:
            self.l = pt
            self.c = 0
        else:
            self.c += 1
        return (self.l, self.c)

    def receive(self, msg_l, msg_c):
        pt = int(time.time() * 1000)
        if pt > self.l and pt > msg_l:
            self.l = pt
            self.c = 0
        elif msg_l > self.l:
            self.l = msg_l
            self.c = msg_c + 1
        elif self.l == msg_l:
            self.c = max(self.c, msg_c) + 1
        else:
            self.c += 1
        return (self.l, self.c)
```

---

## 실무 심화 포인트

- **etcd 운영**: WAL + snapshot 디스크를 전용 SSD에 분리. `etcd --data-dir`을 NVMe에. fsync p99 > 25ms이면 노드 교체 고려
- **Kafka 파티션 리더 편중**: `kafka.server:type=ReplicaManager,name=LeaderCount` 모니터링. `kafka-leader-election.sh --election-type PREFERRED`로 재조정
- **분산 추적의 인과성**: OpenTelemetry의 W3C Trace Context (`traceparent` 헤더)로 서비스 간 인과관계 전파
- **Clock Skew 탐지**: 분산 로그 분석 시 타임스탬프 역전 현상이 보이면 NTP 동기화 문제. 로그에 logical clock도 포함 권장
- **Read-Your-Writes 일관성**: 사용자가 자신의 쓰기는 반드시 읽어야 함. Primary에서 읽거나 세션에 마지막 쓰기 LSN 저장 후 replica가 따라잡을 때까지 대기
- **Fencing Token**: 리더 선출 후 구 리더의 좀비 쓰기 방지. Chubby/etcd의 lease + 단조 증가 토큰으로 스토리지에서 구 토큰 거부
