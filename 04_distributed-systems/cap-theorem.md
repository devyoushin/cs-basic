# CAP 정리 & 분산 시스템 기초

## CAP 정리

분산 시스템은 세 가지 속성 중 **최대 2가지**만 보장 가능:

```
         C (Consistency)
        / \
       /   \
      /     \
     /  CAP  \
    /  Theorem \
   /____________\
A (Availability)  P (Partition Tolerance)
```

| 속성 | 설명 |
|------|------|
| **C**onsistency | 모든 노드에서 같은 시점에 같은 데이터를 읽음 |
| **A**vailability | 모든 요청이 응답을 받음 (성공/실패) |
| **P**artition Tolerance | 네트워크 분리가 발생해도 시스템 동작 |

**현실**: 네트워크 파티션은 반드시 발생 → P는 필수 → **CP 또는 AP 선택**

| 선택 | 특징 | 예시 |
|------|------|------|
| CP | 일관성 보장, 파티션 시 가용성 포기 | ZooKeeper, etcd, HBase |
| AP | 가용성 보장, 파티션 시 일관성 포기 | Cassandra, DynamoDB, CouchDB |

---

## BASE vs ACID

| 속성 | ACID (관계형DB) | BASE (NoSQL) |
|------|---------------|-------------|
| 기본 가정 | 강한 일관성 | 약한 일관성 |
| 가용성 | 일관성 우선 | 가용성 우선 |
| 모델 | 즉각적 일관성 | 결과적 일관성 |

**BASE**:
- **B**asically **A**vailable: 기본적으로 가용
- **S**oft state: 상태가 변할 수 있음
- **E**ventually consistent: 결국에는 일관성 달성

---

## 결과적 일관성 (Eventual Consistency)

```
시간 →
노드A: [v1] → [v2] (업데이트)
노드B: [v1]  →  [v1]  →  [v2] (동기화 지연 후 최신 반영)
```

- 잠시 다른 값을 보여주지만 결국 수렴
- DNS, 쇼핑카트, SNS 좋아요 카운트 등에서 허용 가능

---

## 복제 (Replication)

### 리더-팔로워 (Master-Replica)
```
Writer → Leader → Follower1
                → Follower2
Reader          ← Follower1 (읽기 분산)
```

- **동기 복제**: 리더가 팔로워 확인 후 커밋. 강한 일관성, 쓰기 지연
- **비동기 복제**: 리더가 즉시 커밋. 빠르지만 팔로워 데이터 지연 가능

### 멀티 마스터
- 여러 노드에서 동시에 쓰기 가능
- 충돌 해결 필요 (Last-Write-Wins, 벡터 클락 등)

---

## 합의 알고리즘 (Consensus)

분산 시스템에서 모든 노드가 같은 값에 동의하는 방법.

### Raft
- etcd, Consul 사용
- **Leader Election**: 리더 없으면 선출 과정
- **Log Replication**: 리더가 로그를 팔로워에 복제
- 과반수(Quorum) 동의 시 커밋

### Paxos
- Raft의 전신. 이해하기 복잡
- ZooKeeper의 ZAB(ZooKeeper Atomic Broadcast)은 Paxos 변형

### Quorum (과반수)
```
노드 N개에서 쓰기(W) + 읽기(R) > N이면 강한 일관성 보장
예: N=3, W=2, R=2 → W+R=4 > 3 → 일관성 보장
```

---

## 분산 트랜잭션

### 2PC (Two-Phase Commit)
```
Phase 1 (Prepare): Coordinator → 모든 참가자: "준비됐나?"
Phase 2 (Commit):  Coordinator → 모든 참가자: "커밋해"
```

- 모든 참가자가 준비돼야 커밋
- Coordinator 장애 시 블로킹 문제
- 마이크로서비스에서는 피하는 경향

### Saga 패턴
- 각 서비스의 로컬 트랜잭션 + 실패 시 보상 트랜잭션
- **Choreography**: 이벤트 기반으로 서비스가 서로 통신
- **Orchestration**: 중앙 Saga Orchestrator가 조율

```
주문생성 → 결제처리 → 재고감소 → 배송시작
     ↓ 실패 시 보상
주문취소 ← 결제취소 ← 재고복구
```

---

## 실무 포인트

- **etcd**: Kubernetes의 모든 상태 저장소. CP 시스템. 쿼럼 유지 위해 홀수 노드(3, 5)
- **ZooKeeper**: 분산 코디네이션. Kafka, HBase에서 사용. 역시 CP
- **Cassandra**: AP 시스템. `QUORUM` 일관성 레벨로 강도 조절 가능
- **Kafka 파티션**: 메시지 순서는 파티션 내에서만 보장. 전체 순서 필요 시 단일 파티션
- **분산 ID 생성**: UUID, Snowflake(Twitter), ULID — 단조 증가 + 분산 생성
- **벡터 클락**: 인과관계 추적에 사용. 이벤트 A가 B보다 먼저임을 증명
