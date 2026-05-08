# Kubernetes 내부 동작 완전 분석 (Senior Level)

> kube-proxy, etcd, 스케줄러, CNI, CRI — 각 컴포넌트가 어떻게 동작하는지 커널 레벨까지

---

## 1. Kubernetes 아키텍처 재조명

```
Control Plane                          Worker Node
┌──────────────────────────────┐       ┌──────────────────────────────┐
│  kube-apiserver              │       │  kubelet                     │
│    └─ 모든 컴포넌트의 허브    │◀─────▶│    └─ CRI → containerd/cri-o │
│  etcd                        │       │  kube-proxy                  │
│    └─ 상태 저장소 (Raft)      │       │    └─ iptables / IPVS        │
│  kube-scheduler              │       │  CNI plugin                  │
│    └─ Pod 배치 결정           │       │    └─ 네트워크 설정           │
│  kube-controller-manager     │       │  cAdvisor                    │
│    └─ 상태 조정 루프          │       │    └─ 리소스 메트릭 수집      │
└──────────────────────────────┘       └──────────────────────────────┘
```

### Watch 메커니즘 (핵심 패턴)

```
모든 컴포넌트는 kube-apiserver를 "Watch"한다:

kubelet:
  GET /api/v1/pods?watch=true&fieldSelector=spec.nodeName=node1
  → Long-polling HTTP 연결 유지
  → Pod 변경 이벤트 스트리밍 수신

이벤트 타입: ADDED / MODIFIED / DELETED

etcd Watch:
  kube-apiserver가 etcd의 /registry/** 를 Watch
  → 변경 발생 시 연결된 모든 클라이언트에 fan-out

ResourceVersion:
  각 오브젝트에 단조 증가하는 버전 번호
  Watch 재연결 시 ?resourceVersion=12345 → 해당 버전 이후 이벤트만 수신
  → 재연결 시 이벤트 누락 없음 보장
```

---

## 2. etcd 내부 동작 (Raft 기반)

### Raft 합의 알고리즘

```
etcd 클러스터 (보통 3 또는 5 노드):

[Leader]  ←──── Heartbeat ────→  [Follower]
    │                                  │
    │ 1. 클라이언트 쓰기 요청            │
    │ 2. Log Entry 생성                 │
    │ 3. Follower에게 AppendEntries RPC │──▶ [Follower]
    │ 4. 과반수(Quorum) ACK 수신         │
    │ 5. 커밋 → 상태 머신 적용           │
    │ 6. 클라이언트에 응답               │

Quorum = (N/2) + 1
  3노드: 2개 동의 필요
  5노드: 3개 동의 필요

Leader Election:
  Heartbeat 타임아웃 → Candidate 상태
  RequestVote RPC 전송 → 과반수 투표 → Leader 당선
  선거 타임아웃: 랜덤 150~300ms (스플릿 브레인 방지)
```

### etcd의 MVCC 스토리지

```
etcd는 키-값에 대한 다중 버전을 유지:

key=/registry/pods/default/my-pod
  revision=100: {spec: ...v1}
  revision=150: {spec: ...v2}
  revision=200: (deleted)

Compaction:
  오래된 revision 정리 (기본 5분 또는 5만 revision)
  너무 빠른 compaction → Watch 재연결 시 410 Gone 에러

스토리지 백엔드: bbolt (B-Tree 기반 임베디드 DB)
  /var/lib/etcd/member/snap/db 파일 하나로 관리

etcd 성능 병목:
  - 디스크 fsync 레이턴시 (SSD 필수, 권장 <10ms)
  - 네트워크 레이턴시 (노드 간 RTT < 1ms 권장)
  - 오브젝트 수 증가에 따른 DB 크기 (기본 8GB 제한)
```

```bash
# etcd 상태 확인
ETCDCTL_API=3 etcdctl endpoint status --cluster -w table

# 클러스터 건강도
ETCDCTL_API=3 etcdctl endpoint health --cluster

# DB 크기 및 사용량
ETCDCTL_API=3 etcdctl endpoint status | grep -i db

# 특정 키 조회
ETCDCTL_API=3 etcdctl get /registry/pods/default/my-pod

# Watch (실시간)
ETCDCTL_API=3 etcdctl watch /registry/pods/ --prefix

# Compaction 수동 실행
ETCDCTL_API=3 etcdctl compact $(ETCDCTL_API=3 etcdctl endpoint status --write-out json | jq '.[0].Status.header.revision')

# Defrag (DB 파일 단편화 제거)
ETCDCTL_API=3 etcdctl defrag
```

### etcd 장애 시나리오

```
시나리오 1: 과반수 노드 다운
  3노드 중 2노드 다운 → 쓰기 불가 (읽기는 가능)
  → API 서버는 503 반환 / 새 Pod 스케줄링 불가
  → 기존 실행 중인 Pod는 계속 동작 (kubelet은 독립적)

시나리오 2: DB 크기 한계 도달
  증상: "etcdserver: mvcc: database space exceeded"
  → 모든 쓰기 거부
  해결:
    1. ETCDCTL_API=3 etcdctl compact <revision>
    2. ETCDCTL_API=3 etcdctl defrag
    3. --quota-backend-bytes 증가 (최대 8GB)

시나리오 3: 클러스터 스플릿 브레인
  네트워크 파티션으로 양쪽에 Leader 선출
  Raft 보장: 과반수 없는 쪽은 쓰기 거부 → 데이터 일관성 유지
```

---

## 3. kube-scheduler 내부

### 스케줄링 사이클

```
Pod 생성
  │
  ▼
[Scheduling Queue]  ← PriorityQueue 정렬
  │
  ▼
[Filtering Phase]   (모든 노드에 대해 병렬 실행)
  │  NodeUnschedulable, NodeAffinity, Taints/Tolerations
  │  ResourceFit, PodTopologySpread, VolumeBinding ...
  ▼
[Scoring Phase]     (필터 통과 노드에 대해 점수 계산)
  │  LeastAllocated, BalancedAllocation, NodeAffinity
  │  InterPodAffinity, ImageLocality ...
  ▼
[최고 점수 노드 선택]
  │
  ▼
[Binding]           → kube-apiserver에 NodeBinding 오브젝트 생성
  │
  ▼
kubelet이 Pod 시작
```

### 스케줄러 확장: Scheduling Framework

```
v1.19+에서 Plugin 방식으로 전환

플러그인 확장 포인트:
  PreFilter  → 전처리, 사이클 시작 시 단 한 번 실행
  Filter     → 노드 필터링 (Predicate)
  PostFilter → 필터 실패 시 (Preemption 로직)
  PreScore   → 스코어링 전 처리
  Score      → 노드 점수 계산 (Priority)
  Reserve    → 리소스 예약 (Volume binding 등)
  Permit     → 바인딩 전 게이트 (대기/거부/승인)
  Bind       → 실제 바인딩

커스텀 스케줄러:
  - 별도 스케줄러 프로세스 실행
  - Pod spec.schedulerName으로 지정
  - 예: GPU 스케줄러, 배치 워크로드 특화 스케줄러
```

### Preemption (선점)

```
High Priority Pod가 스케줄 불가 → Preemption 시작:
  1. 낮은 우선순위 Pod를 Evict하면 스케줄 가능한 노드 탐색
  2. Evict 대상 Pod에 graceful termination 시작
  3. NominatedNode 필드에 노드 기록
  4. 대상 Pod 종료 후 High Priority Pod 스케줄링

PriorityClass:
  systemClusterCritical: 2000000000 (최우선)
  systemNodeCritical:    2000001000
  사용자 정의: 1 ~ 1000000000
```

```bash
# 스케줄러 이벤트 확인
kubectl get events --field-selector reason=FailedScheduling -A

# Pod 스케줄링 상세 이유
kubectl describe pod <pod-name> | grep -A10 Events

# 노드별 할당 리소스
kubectl describe nodes | grep -A5 "Allocated resources"

# 스케줄러 로그 (verbose)
kubectl logs -n kube-system deployment/kube-scheduler | grep -i "Attempting to schedule"
```

---

## 4. kube-proxy 내부 동작

### iptables 모드 (기본)

```
Service 생성 시 kube-proxy가 iptables 규칙 생성:

패킷 흐름:
  ClusterIP:Port 수신
  → KUBE-SERVICES 체인
    → KUBE-SVC-XXXXXX 체인 (Service별)
      → 확률적 DNAT (각 Endpoint로)
        KUBE-SEP-XXXXXX: DNAT → PodIP:Port

예시 규칙 (3개 Endpoint):
  -A KUBE-SVC-XYZ -m statistic --mode random --probability 0.33 -j KUBE-SEP-1
  -A KUBE-SVC-XYZ -m statistic --mode random --probability 0.5  -j KUBE-SEP-2
  -A KUBE-SVC-XYZ -j KUBE-SEP-3
  (첫 번째: 33%, 두 번째: 나머지의 50%=33%, 세 번째: 나머지=33%)
```

```bash
# iptables 규칙 확인
iptables -t nat -L KUBE-SERVICES -n | head -30
iptables -t nat -L KUBE-SVC-<hash> -n

# Service 이름으로 관련 체인 찾기
iptables -t nat -L -n | grep <service-name>

# 규칙 수 확인 (규모 파악)
iptables -t nat -L | wc -l
```

**iptables 모드의 한계:**
```
Service 수 증가 → iptables 규칙 수 폭발
  100 Services, 10 Endpoints = ~10,000 iptables 규칙
  1000 Services = ~100,000 규칙 → 규칙 순회 O(n) → 레이턴시 증가

순차 순회: 패킷마다 전체 규칙 체인 탐색 (해시 테이블 아님)
```

### IPVS 모드 (대규모 클러스터 권장)

```
IPVS (IP Virtual Server): 커널 레벨 L4 로드 밸런서

iptables vs IPVS:
  iptables: O(n) 선형 탐색, 규칙 업데이트 시 전체 재작성
  IPVS: O(1) 해시 테이블 탐색, 증분 업데이트

지원 알고리즘:
  rr (Round Robin), lc (Least Connection),
  dh (Destination Hash), sh (Source Hash),
  sed (Shortest Expected Delay), nq (Never Queue)
```

```bash
# IPVS 모드 활성화 (kube-proxy configmap)
kubectl edit configmap kube-proxy -n kube-system
# mode: "ipvs"
# ipvs.scheduler: "rr"

# IPVS 규칙 확인
ipvsadm -Ln

# IPVS 통계
ipvsadm -Ln --stats

# 특정 VIP 확인
ipvsadm -Ln | grep -A5 <ClusterIP>
```

### NodePort와 외부 트래픽

```
NodePort 흐름:
  외부 클라이언트 → Node:30080
  → iptables PREROUTING
  → DNAT → PodIP:8080 (다른 노드의 Pod 포함)
  → Pod 처리
  → SNAT (externalTrafficPolicy: Cluster일 때)
  → 클라이언트에 응답

externalTrafficPolicy: Local
  → 해당 노드의 Pod에만 라우팅
  → SNAT 없음 → 실제 클라이언트 IP 보존
  → 단: 해당 노드에 Pod 없으면 연결 거부

LoadBalancer Service:
  Cloud LB → NodePort → 클러스터 내 라우팅
  AWS: NLB(Network LB) 또는 CLB
  externalTrafficPolicy: Local + NLB = 실제 IP 보존 가능
```

---

## 5. CNI (Container Network Interface) 내부

### CNI 동작 원리

```
Pod 생성 시 kubelet → CRI(containerd) → CNI 플러그인 호출

CNI 호출 순서:
  1. containerd가 네트워크 네임스페이스 생성
     /proc/<pid>/ns/net (또는 /var/run/netns/<name>)

  2. CNI 플러그인 실행 (환경변수 전달):
     CNI_COMMAND=ADD
     CNI_CONTAINERID=<container-id>
     CNI_NETNS=/var/run/netns/cni-xxx
     CNI_IFNAME=eth0

  3. CNI 플러그인 동작:
     - veth pair 생성 (eth0 in pod ↔ vethXXX on host)
     - Pod에 IP 할당 (IPAM)
     - 라우팅 규칙 설정
     - iptables 규칙 추가 (필요 시)

  4. JSON 응답으로 결과 반환
     {"cniVersion": "0.4.0", "interfaces": [...], "ips": [...]}
```

### Flannel (Overlay 네트워크)

```
각 노드에 서브넷 할당 (예: /24):
  Node1: 10.244.1.0/24
  Node2: 10.244.2.0/24

Pod 간 통신 (다른 노드):
  Pod(10.244.1.5) → veth → flannel0 (TUN/TAP 또는 VXLAN)
  → VXLAN 캡슐화: 원본 패킷을 UDP 8472로 감쌈
  → Node2의 flannel이 디캡슐화 → Pod(10.244.2.7)

VXLAN 오버헤드:
  원본 패킷 + VXLAN 헤더(50byte) → MTU 고려 필요 (MTU 1450 설정)

etcd에 서브넷 정보 저장:
  /coreos.com/network/subnets/10.244.1.0-24 → Node1 IP
```

### Calico (L3 라우팅, 오버레이 없음)

```
BGP 기반 라우팅:
  각 노드가 BGP 피어로 동작
  Pod IP를 BGP로 전파 → 오버레이 없이 직접 라우팅

패킷 흐름:
  Pod(10.244.1.5) → veth → 호스트 라우팅 테이블
  → BGP 경로로 Node2에 직접 전달 (캡슐화 없음)

NetworkPolicy:
  Calico는 iptables 또는 eBPF로 NetworkPolicy 구현
  - iptables: 기존 방식, 규모 확장 시 성능 이슈
  - eBPF: 커널 내 실행, 높은 성능

IP-in-IP 모드 (클라우드 환경):
  BGP 직접 라우팅이 불가한 환경(AWS VPC 등)에서
  IP-in-IP 캡슐화로 오버레이 구성 가능
```

```bash
# 현재 CNI 플러그인 확인
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf

# Pod의 네트워크 인터페이스
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- ip route

# 호스트에서 veth pair 확인
ip link show type veth

# Calico 피어 상태
calicoctl node status

# CNI 로그
journalctl -u kubelet | grep -i cni
```

---

## 6. CRI (Container Runtime Interface) 내부

### CRI 아키텍처

```
kubelet
  │  gRPC
  ▼
CRI shim (containerd, cri-o)
  │
  ▼
OCI Runtime (runc, gVisor, kata-containers)
  │
  ▼
Linux Kernel (namespace, cgroup)
```

### containerd 내부

```
containerd 컴포넌트:
  - containerd-shim-runc-v2: 각 컨테이너마다 별도 shim 프로세스
    → kubelet 재시작 시 컨테이너 유지 가능
    → shim이 컨테이너 stdio, exit status 관리

이미지 레이어:
  Content Store: OCI 레이어 블롭 저장 (/var/lib/containerd/io.containerd.content.v1.content)
  Snapshotter: OverlayFS 레이어 마운트 관리
  Metadata: boltdb로 이미지/컨테이너 메타데이터 저장

컨테이너 생성 흐름:
  kubelet → CRI gRPC → containerd
  → RunPodSandbox (pause 컨테이너, 네트워크 네임스페이스 생성)
  → PullImage (레지스트리에서 이미지 pull)
  → CreateContainer (OverlayFS 마운트, OCI spec 생성)
  → StartContainer → runc create → runc start
```

```bash
# containerd 네임스페이스 목록 (k8s용은 "k8s.io")
ctr namespace ls

# Pod/컨테이너 목록
crictl pods
crictl ps

# 이미지 목록
crictl images

# 컨테이너 로그
crictl logs <container-id>

# 컨테이너 상세 정보 (OCI spec 포함)
crictl inspect <container-id>

# 컨테이너 내부 exec
crictl exec -it <container-id> /bin/sh

# containerd 직접 접근
ctr -n k8s.io containers ls
ctr -n k8s.io snapshots ls
```

### Pause 컨테이너 역할

```
모든 Pod에는 pause 컨테이너(infra container)가 존재:

역할:
  1. Linux 네임스페이스 홀더
     - Network namespace → Pod 내 모든 컨테이너가 공유
     - IPC namespace → 컨테이너 간 공유 메모리
     - PID namespace (설정 시)

  2. Init process (PID 1) 역할
     - 좀비 프로세스 수거 (wait())
     - 시그널 전달

pause 컨테이너 없이 앱 컨테이너가 죽으면 → 네트워크 설정 사라짐
pause 컨테이너가 죽으면 → Pod 전체 재시작
```

---

## 7. 리소스 관리 내부

### QoS 클래스

```
Guaranteed:
  requests == limits (CPU + Memory 모두)
  → OOM Killer 우선순위 가장 낮음 (마지막에 kill)

Burstable:
  requests < limits 또는 일부만 설정
  → 중간 우선순위

BestEffort:
  requests, limits 모두 미설정
  → OOM Killer 최우선 대상

OOM Score 계산:
  /proc/<pid>/oom_score_adj
  BestEffort: 1000 (최우선 kill)
  Burstable:  요청 비율 기반 (높을수록 먼저 kill)
  Guaranteed: -997 (거의 kill 안 됨)
```

```bash
# Pod의 QoS 클래스 확인
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# OOM Score 확인
cat /proc/$(pgrep <process>)/oom_score_adj

# 노드에서 OOM 이벤트 확인
dmesg | grep -i "oom\|killed"
kubectl describe node <node> | grep -i "oom\|evict"
```

### cgroup v2와 리소스 제한

```
cgroup v2 계층:
  /sys/fs/cgroup/
    kubepods.slice/
      burstable.slice/
        pod<uid>.slice/
          <container-id>/
            memory.max        ← limits.memory
            memory.min        ← requests.memory (보장)
            cpu.max           ← limits.cpu (quota/period)
            cpu.weight        ← requests.cpu (상대적 가중치)

CPU throttling 확인:
  /sys/fs/cgroup/.../cpu.stat
  nr_throttled: 스로틀된 횟수
  throttled_usec: 스로틀된 총 시간
```

```bash
# CPU throttling 확인
cat /sys/fs/cgroup/kubepods.slice/*/cpu.stat | grep throttled

# 컨테이너 cgroup 찾기
systemd-cgls | grep <container-id>

# Pod 리소스 사용량
kubectl top pod <pod-name> --containers

# 노드 리소스 압박 상태
kubectl describe node <node> | grep -A5 "Conditions:"
# MemoryPressure, DiskPressure, PIDPressure
```

---

## 8. 실무 심화 포인트 & 면접 대비

### "kube-proxy iptables와 IPVS의 차이는?"
```
iptables:
  규칙 수: O(Services × Endpoints)
  탐색: O(n) 순차
  업데이트: 전체 체인 재작성 (lock 필요)
  규모: ~1000 Services까지 적합

IPVS:
  해시 테이블 기반 O(1) 탐색
  증분 업데이트 가능
  더 많은 LB 알고리즘 지원
  규모: 10,000+ Services 가능
  주의: conntrack 의존도 높음
```

### "etcd 3노드 중 1노드 다운 시 어떻게 되나?"
```
쓰기: 가능 (2/3 quorum 만족)
읽기: 가능
새 Leader 선출: 불필요 (기존 Leader 유지)
1노드 복구 전 또 1노드 다운 → 쓰기 불가
→ etcd는 홀수 노드로 구성해야 함 (2노드는 Quorum 이점 없음)
```

### "Pod가 Pending 상태에서 멈춘 이유 진단 순서"
```
1. kubectl describe pod → Events 섹션 확인
   - "Insufficient cpu/memory" → 리소스 부족
   - "node(s) had taints" → Toleration 누락
   - "pod has unbound PVC" → PV 미생성
   - "Unschedulable" → Node Affinity/Anti-affinity

2. kubectl get nodes → Ready 상태 확인

3. kubectl describe nodes → Allocated resources 확인
   CPU/Memory 요청량 vs 실제 available 비교

4. 스케줄러 로그
   kubectl logs -n kube-system deploy/kube-scheduler
```

### "컨테이너가 OOM Kill되는데 limits 내에서 죽는 이유?"
```
가능한 원인:
  1. limits.memory에 도달 → 컨테이너 cgroup OOM
  2. 노드 전체 메모리 부족 → 노드 OOM Killer 작동
     (BestEffort/Burstable Pod가 먼저 kill)
  3. HugePage 사용 시 일반 memory limits 우회

진단:
  kubectl describe pod → "OOMKilled" 확인
  dmesg | grep "oom-kill" → 어느 프로세스가 kill 됐는지
  /proc/<pid>/oom_score_adj 확인
```

### "NetworkPolicy가 적용되지 않는 경우"
```
NetworkPolicy는 CNI 플러그인이 구현:
  - Flannel: 기본적으로 NetworkPolicy 미지원
  - Calico, Cilium, WeaveNet: 지원

확인 순서:
  1. CNI 플러그인 확인: kubectl get pods -n kube-system
  2. NetworkPolicy 오브젝트 존재 확인
  3. podSelector 라벨 매칭 확인
     kubectl get networkpolicy -o yaml
  4. Calico: calicoctl get networkpolicy

일반적인 실수:
  - podSelector: {} → 네임스페이스의 모든 Pod에 적용
  - ingress/egress 둘 다 설정해야 양방향 제어
  - 같은 네임스페이스 내 Pod도 namespaceSelector 필요
```

| 질문 | 핵심 답변 |
|------|-----------|
| pause 컨테이너 목적 | 네트워크/IPC 네임스페이스 홀더, 좀비 수거 |
| etcd Quorum이란 | (N/2)+1 노드 동의 필요. 3노드=2, 5노드=3 |
| CNI와 CSI 차이 | CNI=네트워크 플러그인, CSI=스토리지 플러그인 |
| Deployment vs StatefulSet | 순서/ID 보장 여부. StatefulSet은 Pod당 고유 ID/볼륨 |
| ResourceVersion 용도 | 낙관적 잠금 + Watch 재연결 시 이벤트 연속성 보장 |
