# Kubernetes 기초

## K8s 아키텍처: Control Plane + Data Plane

```
Control Plane (마스터)                    Data Plane (워커 노드)
┌────────────────────────────┐           ┌─────────────────────────┐
│  kube-apiserver            │           │  kubelet                │
│  (모든 요청의 진입점)        │◄─────────►│  (Node Agent)           │
│                            │           │                         │
│  etcd                      │           │  kube-proxy             │
│  (클러스터 상태 저장소)      │           │  (iptables/ipvs 관리)   │
│                            │           │                         │
│  kube-scheduler            │           │  Container Runtime      │
│  (Pod 배치 결정)            │           │  (containerd/CRI-O)     │
│                            │           │                         │
│  kube-controller-manager   │           │  Pods                   │
│  (상태 조정 루프)           │           │  └── Containers         │
└────────────────────────────┘           └─────────────────────────┘
```

---

## etcd: 클러스터의 진실 공급원

모든 K8s 오브젝트 (Pod, Service, ConfigMap 등)가 etcd에 저장됨.

```
etcd = 분산 Key-Value 저장소
키: /registry/pods/default/nginx-pod
값: Pod 오브젝트 JSON (protobuf 인코딩)
```

- Raft 합의 알고리즘으로 고가용성 (3 or 5 노드 권장)
- kube-apiserver만 etcd에 직접 접근 (다른 컴포넌트는 API Server 통해 접근)
- etcd 장애 = 클러스터 불능. 백업 필수 (`etcdctl snapshot save`)

---

## API Server & 요청 처리 흐름

`kubectl apply -f pod.yaml` 실행 시:

```
kubectl → API Server
    ① Authentication (인증): 누구인가? (cert, Bearer token, OIDC)
    ② Authorization (인가): 무엇을 할 수 있는가? (RBAC)
    ③ Admission Control: 정책 검사 (MutatingWebhook, ValidatingWebhook)
       - LimitRanger: 리소스 기본값 주입
       - ResourceQuota: 네임스페이스 쿼터 검사
       - PodSecurity: 보안 정책 검사
    ④ Validation: 오브젝트 스키마 검증
    ⑤ Persist to etcd
    ⑥ 응답 반환
```

---

## Controller Manager: 조정 루프 (Reconciliation Loop)

K8s의 핵심 철학: **선언적 상태(Desired State) vs 현재 상태(Current State) 조정**

```
while true:
    desired = get from etcd (spec)
    current = observe actual state
    if desired != current:
        take action to converge
    sleep(resync_period)
```

주요 Controller:
- **ReplicaSet Controller**: `replicas: 3` 선언 → 현재 2개면 1개 생성
- **Deployment Controller**: RollingUpdate 전략 실행
- **Endpoint Controller**: Service의 Endpoints 오브젝트 업데이트
- **Node Controller**: 노드 장애 감지 → Pod Eviction

---

## Scheduler: Pod 배치 알고리즘

Pod 생성 시 `nodeName` 미지정 → Scheduler가 노드 선택.

```
① Filtering (필터링): 불가능한 노드 제거
   - NodeResourcesFit: 요청 리소스 수용 가능한지
   - NodeAffinity: affinity 규칙 충족하는지
   - TaintToleration: Taint 허용되는지
   - PodTopologySpread: 토폴로지 분산 규칙

② Scoring (점수화): 남은 노드 중 최적 선택
   - LeastAllocated: 가장 여유 있는 노드 우선
   - ImageLocality: 이미지가 이미 있는 노드 우선
   - NodeAffinity: preferred affinity 점수

③ 최고 점수 노드에 Bind (etcd에 nodeName 업데이트)
④ kubelet이 Pod 생성
```

---

## Pod: 스케줄링의 기본 단위

Pod는 **하나 이상의 컨테이너 + 공유 네트워크 + 공유 스토리지**.

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:          # 스케줄링 기준 (Scheduler가 사용)
        cpu: "250m"      # 0.25 CPU
        memory: "256Mi"
      limits:            # 하드 한도 (cgroup에 설정)
        cpu: "500m"
        memory: "512Mi"
    livenessProbe:       # 살아있는지 (실패 시 재시작)
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:      # 트래픽 받을 준비됐는지 (실패 시 Endpoint 제거)
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

### QoS (Quality of Service) 클래스

| 클래스 | 조건 | OOM Kill 우선순위 |
|--------|------|----------------|
| **Guaranteed** | requests == limits (모든 컨테이너) | 마지막 (가장 보호됨) |
| **Burstable** | requests < limits 또는 일부만 설정 | 중간 |
| **BestEffort** | requests/limits 없음 | 가장 먼저 종료 |

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

### Pod 생명주기

```
Pending → ContainerCreating → Running → Succeeded/Failed
  ↑ 노드 배정 전         ↑ 이미지 풀    ↑ 실행 중

재시작 정책 (restartPolicy):
- Always: 항상 재시작 (Deployment 기본)
- OnFailure: 실패 시만 (Job)
- Never: 재시작 없음
```

---

## Deployment & ReplicaSet

```
Deployment
  └── ReplicaSet (v1, 이전 버전)
  └── ReplicaSet (v2, 현재)   ← pods 관리
        ├── Pod-1
        ├── Pod-2
        └── Pod-3
```

### Rolling Update 메커니즘

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # 동시에 추가 생성 가능한 Pod 수 (replicas 초과 허용)
    maxUnavailable: 0  # 동시에 중단 가능한 Pod 수 (0 = 항상 replicas 유지)
```

```
replicas: 3, maxSurge=1, maxUnavailable=0:

Step 1: [v1][v1][v1]         → [v1][v1][v1][v2]  (4개, maxSurge=1)
Step 2: [v1][v1][v1][v2]     → [v1][v1][v2]      (v1 하나 제거)
Step 3: [v1][v1][v2]         → [v1][v1][v2][v2]
Step 4: [v1][v1][v2][v2]     → [v1][v2][v2]
...
```

readinessProbe 통과 후에만 트래픽 전달 & 이전 Pod 제거 진행 → 무중단 배포 가능.

---

## Service: 안정적인 엔드포인트

Pod는 재시작 시 IP가 바뀜. Service는 **고정 ClusterIP**로 Pod 집합에 대한 안정적 접근 제공.

```
Service (ClusterIP: 10.96.100.50)
  └── Selector: app=nginx
        ├── Pod-1 (172.16.1.1)
        ├── Pod-2 (172.16.1.2)
        └── Pod-3 (172.16.2.1)
```

### kube-proxy의 실제 구현

```
Service IP(10.96.100.50:80) → 실제 Pod IP로 전달 방법:
```

**iptables 모드** (기본):
```bash
# PREROUTING: Service IP로 오는 패킷을 랜덤하게 Pod로 DNAT
-A KUBE-SERVICES -d 10.96.100.50/32 -p tcp --dport 80 -j KUBE-SVC-XXXX

# 1/3 확률로 Pod-1로
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.33 -j KUBE-SEP-AAA
# 1/2 확률로 Pod-2로 (남은 것 중)
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.50 -j KUBE-SEP-BBB
# 나머지는 Pod-3로
-A KUBE-SVC-XXXX -j KUBE-SEP-CCC

-A KUBE-SEP-AAA -p tcp -j DNAT --to-destination 172.16.1.1:80
```

**IPVS 모드** (대규모 클러스터 권장):
- LVS(Linux Virtual Server) 사용 → 커널 내 해시 테이블
- O(1) 규칙 조회 (iptables는 O(n)) → Service 수천 개일 때 성능 차이 큼
- 다양한 LB 알고리즘 지원 (RR, LC, SH, ...)

```bash
kubectl edit configmap -n kube-system kube-proxy
# mode: "ipvs"
ipvsadm -ln  # IPVS 규칙 확인
```

### Service 타입

| 타입 | 설명 | 사용 |
|------|------|------|
| ClusterIP | 클러스터 내부 전용 (기본) | 서비스 간 통신 |
| NodePort | 모든 노드의 특정 포트 노출 (30000-32767) | 외부 테스트 |
| LoadBalancer | 클라우드 LB 프로비저닝 | 운영 외부 노출 |
| ExternalName | CNAME으로 외부 서비스 매핑 | 외부 DB 추상화 |

---

## Ingress: L7 라우팅

```
외부 → LoadBalancer → Ingress Controller (nginx/traefik) → Service → Pod

Ingress 규칙:
  /api/* → api-service:8080
  /web/* → web-service:3000
  app.example.com → app-service:80
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["app.example.com"]
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## ConfigMap & Secret

```yaml
# ConfigMap: 설정값 (평문)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  DB_HOST: "postgres-service"

# Secret: 민감 데이터 (base64 인코딩, etcd 암호화 필요)
apiVersion: v1
kind: Secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=   # "password" base64
```

**주의**: Secret은 base64 인코딩이지 암호화가 아님.
etcd 암호화 활성화 필수: `EncryptionConfiguration` → AES-CBC or AES-GCM 사용.

---

## RBAC (Role-Based Access Control)

```
Subject (누가)     Verb (무엇을)       Resource (어디에)
ServiceAccount → get, list, watch → pods, deployments

Role: 네임스페이스 범위
ClusterRole: 클러스터 전체 범위
```

```yaml
# Role: default 네임스페이스에서 Pod 읽기만 허용
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding: 특정 ServiceAccount에 Role 부여
kind: RoleBinding
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
```

**최소 권한 원칙**: ServiceAccount는 기본적으로 API 접근 권한 없음. 필요한 것만 명시적으로 부여.

---

## K8s 네트워킹 모델 (CNI)

K8s 네트워킹 3대 원칙:
1. 모든 Pod는 NAT 없이 다른 Pod와 통신 가능
2. 모든 Node는 NAT 없이 모든 Pod와 통신 가능
3. Pod가 자기 자신을 인식하는 IP = 다른 Pod가 인식하는 IP

이를 구현하는 CNI(Container Network Interface) 플러그인:

| CNI | 방식 | 특징 |
|-----|------|------|
| Flannel | VXLAN 오버레이 | 단순, 성능 보통 |
| Calico | BGP 또는 VXLAN | NetworkPolicy 강력, 성능 좋음 |
| Cilium | eBPF | 고성능, 가시성, L7 정책 |
| AWS VPC CNI | 직접 VPC IP 할당 | 오버레이 없음, 성능 최고 (AWS) |

### NetworkPolicy
```yaml
# default: 모든 트래픽 허용
# NetworkPolicy 적용: 명시적으로 허용된 것만 통과

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-netpol
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # frontend Pod만 접근 허용
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres     # postgres만 나가기 허용
    ports:
    - port: 5432
```

---

## 실무 포인트

- **requests ≠ limits 주의**: CPU limits는 스로틀링(느려짐), Memory limits는 OOM Kill. CPU limits를 너무 낮게 잡으면 Throttling으로 레이턴시 급등. `kubectl top pod`으로 실제 사용량 확인 후 설정
- **readinessProbe 필수**: 없으면 컨테이너 시작 즉시 트래픽 인입 → 초기화 안 된 상태에서 502 발생
- **Pod Disruption Budget (PDB)**: 롤링 업데이트/노드 드레인 시 최소 가용 Pod 수 보장
  ```yaml
  spec:
    minAvailable: 2   # 항상 최소 2개 유지
    selector:
      matchLabels:
        app: api
  ```
- **Node Affinity vs Taint/Toleration**: Affinity는 Pod가 노드를 선호, Taint는 노드가 Pod를 거부. GPU 노드에는 Taint로 일반 Pod 배제
- **etcd 백업**: `etcdctl snapshot save` 정기 실행. 클러스터 복구의 유일한 방법
- **kube-proxy IPVS 전환**: Service 500개 이상이면 iptables 규칙 수천 개 → 성능 저하. IPVS로 전환 권장
