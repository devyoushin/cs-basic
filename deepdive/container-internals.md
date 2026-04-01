# 컨테이너 내부 구조 (Senior Level)

## 컨테이너 = Namespace + cgroup + Overlay FS

컨테이너는 새로운 기술이 아님. 리눅스 커널 기능의 조합.

```
Docker run nginx
  ↓
  Linux Namespaces: 격리 (PID, NET, MNT, UTS, IPC, USER)
  cgroups:          리소스 제한 (CPU, Memory, I/O, Network)
  Overlay FS:       레이어드 파일시스템
  seccomp:          syscall 필터링
  capabilities:     root 권한 세분화
```

---

## Linux Namespaces 상세

### Namespace 종류
```bash
# 현재 프로세스의 namespace 확인
ls -la /proc/$$/ns/

# 출력:
# lrwxrwxrwx cgroup -> cgroup:[...]
# lrwxrwxrwx ipc    -> ipc:[...]
# lrwxrwxrwx mnt    -> mnt:[...]
# lrwxrwxrwx net    -> net:[...]
# lrwxrwxrwx pid    -> pid:[...]
# lrwxrwxrwx user   -> user:[...]
# lrwxrwxrwx uts    -> uts:[...]
```

| Namespace | 격리 대상 | 컨테이너 적용 |
|-----------|----------|-------------|
| PID | 프로세스 ID 트리 | 컨테이너 내 PID 1이 실제론 다른 PID |
| NET | 네트워크 스택, 인터페이스, 라우팅 | 가상 이더넷 쌍(veth) |
| MNT | 파일시스템 마운트 포인트 | 컨테이너 루트 파일시스템 격리 |
| UTS | hostname, domainname | 컨테이너별 hostname |
| IPC | System V IPC, POSIX MQ | IPC 격리 |
| USER | UID/GID 매핑 | rootless 컨테이너 |
| cgroup | cgroup 루트 뷰 | |

### PID Namespace 동작
```bash
# 호스트에서 컨테이너 프로세스 확인
ps aux | grep nginx
# 호스트 PID: 12345

# 컨테이너 내부에서
docker exec -it my_nginx ps aux
# 컨테이너 PID: 1   ← 같은 프로세스, 다른 PID 뷰

# unshare로 직접 namespace 생성
unshare --pid --fork --mount-proc /bin/bash
# 이 bash는 자신이 PID 1로 보임
ps aux  # PID 1, 2만 보임
```

### NET Namespace와 veth pair
```bash
# 컨테이너 네트워크 격리 구조
#
# 호스트:
#   docker0(bridge) ─── veth1a ──→ [NET NS] ─── eth0 (컨테이너)
#                    └── veth2a ──→ [NET NS] ─── eth0 (컨테이너)

# 컨테이너의 실제 인터페이스 확인
docker inspect <container_id> | jq '.[].NetworkSettings'

# 호스트에서 veth pair 확인
ip link | grep veth

# 컨테이너 network namespace에 직접 진입
PID=$(docker inspect -f '{{.State.Pid}}' <container_id>)
nsenter --target $PID --net ip addr
nsenter --target $PID --net ss -tulnp
```

---

## cgroups v2 상세

### 계층 구조
```
/sys/fs/cgroup/
├── system.slice/
│   ├── nginx.service/
│   │   ├── cgroup.procs      # 프로세스 PID 목록
│   │   ├── cpu.max           # CPU 제한
│   │   ├── memory.max        # 메모리 제한
│   │   └── io.max            # I/O 제한
│   └── ...
└── kubepods/
    └── pod<uid>/
        └── <container_id>/
```

### CPU 제어
```bash
# cpu.max: "quota period" 형식
# 100ms 주기 중 50ms 사용 = 50% CPU
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max

# cpu.weight: 상대적 가중치 (기본 100)
echo 200 > /sys/fs/cgroup/myapp/cpu.weight

# 실시간 CPU 사용량 확인
cat /sys/fs/cgroup/myapp/cpu.stat
# usage_usec: 총 CPU 사용 시간
# throttled_usec: 쓰로틀된 시간 ← 이 값이 크면 CPU limit 너무 낮음
```

### 메모리 제어
```bash
# 하드 제한 (초과 시 OOM kill)
echo "512M" > /sys/fs/cgroup/myapp/memory.max

# 소프트 제한 (메모리 압박 시 우선 회수)
echo "400M" > /sys/fs/cgroup/myapp/memory.high

# 메모리 사용 현황
cat /sys/fs/cgroup/myapp/memory.stat
# anon: 익명 메모리 (힙, 스택)
# file: 파일 캐시
# slab: 커널 슬랩 캐시

# OOM 이벤트 확인
cat /sys/fs/cgroup/myapp/memory.events
# oom_kill: OOM kill 발생 횟수

# 쿠버네티스에서 컨테이너 cgroup 경로
cat /sys/fs/cgroup/kubepods/pod<uid>/<container>/memory.max
```

### I/O 제어
```bash
# 블록 디바이스 번호 확인
cat /proc/partitions  # major:minor

# I/O 제한 설정 (읽기/쓰기 각각 10MB/s)
echo "8:0 rbps=10485760 wbps=10485760" > /sys/fs/cgroup/myapp/io.max

# I/O 통계
cat /sys/fs/cgroup/myapp/io.stat
```

---

## Overlay Filesystem

컨테이너 레이어드 파일시스템의 핵심.

```
Image Layer (읽기 전용):
  Layer 3: /app/server  (최상위, 가장 최근)
  Layer 2: /usr/local/python
  Layer 1: /etc, /usr, /bin  (base OS)

Container Layer (읽기/쓰기):
  upperdir: 변경사항 (새 파일, 수정된 파일)

Overlay Mount:
  lower=layer3:layer2:layer1, upper=container_rw, merged=/
```

```bash
# 실제 overlay 마운트 확인
mount | grep overlay
# overlay on / type overlay (rw,lowerdir=...,upperdir=...,workdir=...)

# Docker 이미지 레이어 확인
docker history nginx:latest

# 컨테이너 변경사항 확인 (upperdir)
docker diff <container_id>
# A = 추가, C = 변경, D = 삭제

# 레이어 물리적 위치 (기본)
ls /var/lib/docker/overlay2/
```

### Copy-on-Write 성능 영향
```
쓰기 시: 파일을 upperdir로 복사 후 수정 (첫 번째 쓰기만 느림)
읽기 시: 상위 레이어부터 순서대로 탐색

주의:
- 레이어가 많을수록 읽기 탐색 오버헤드 증가 (실무에서 10~15개 이하 권장)
- 컨테이너 내에서 대용량 파일 빈번한 수정 → 볼륨 사용 권장
```

---

## seccomp & capabilities

### Linux Capabilities
root 권한을 세분화한 것. 컨테이너는 기본적으로 일부만 부여.

```bash
# 현재 프로세스의 capabilities
cat /proc/self/status | grep Cap
capsh --decode=<hex>

# 컨테이너 기본 허용 capabilities (Docker)
# CAP_NET_BIND_SERVICE: 1024 이하 포트 바인딩
# CAP_CHOWN, CAP_FOWNER 등

# 모든 capabilities 제거 후 필요한 것만 추가
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

### seccomp (Secure Computing Mode)
syscall 필터링으로 공격 표면 축소:

```json
// Docker 기본 seccomp 프로파일 예시
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["accept", "accept4", "access", "read", "write", ...],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["ptrace", "reboot", "mount"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

```bash
# seccomp 위반 확인
dmesg | grep "audit: type=1326"  # seccomp action

# 앱이 사용하는 syscall 프로파일 생성
strace -f -e trace=all ./myapp 2>&1 | grep -oP 'syscall: \K\w+' | sort -u
```

---

## 쿠버네티스 네트워킹 심화

### Pod 네트워킹 (CNI)
```
Pod A (Node 1)                Pod B (Node 2)
  eth0: 10.244.1.5              eth0: 10.244.2.7
    |                               |
  veth pair                       veth pair
    |                               |
  cni0 (bridge)                   cni0 (bridge)
    |                               |
  flannel.1 (VXLAN)─── 물리 NIC ─── flannel.1
  또는 Calico BGP 라우팅
```

```bash
# Pod의 네트워크 namespace 확인
CONTAINER_ID=$(kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d/ -f3)
PID=$(crictl inspect $CONTAINER_ID | jq '.info.pid')
nsenter --target $PID --net ip route

# 노드의 iptables 규칙 (kube-proxy)
iptables -t nat -L KUBE-SERVICES -n | head -20
iptables -t nat -L KUBE-SVC-<hash> -n  # 특정 서비스

# eBPF 기반 CNI (Cilium) — iptables 없음
cilium status
cilium endpoint list
```

### Service 구현체 (kube-proxy)
```
ClusterIP 서비스:
  Client Pod → 10.96.0.1:80 (ClusterIP)
  ↓ iptables DNAT 규칙
  → 10.244.1.5:8080 (Pod A) or
  → 10.244.2.7:8080 (Pod B)  ← 랜덤 선택

IPVS 모드 (대규모 클러스터):
  iptables 규칙 대신 IPVS 사용 → O(1) 조회 (iptables는 O(n))
  kube-proxy --proxy-mode=ipvs
```

---

## 컨테이너 런타임 계층

```
kubectl → API Server → kubelet
                         ↓
                   CRI (Container Runtime Interface)
                         ↓
               containerd (고수준 런타임)
                         ↓
                    runc (저수준 런타임, OCI 스펙)
                         ↓
                    Linux Kernel (namespace, cgroup)
```

```bash
# containerd 직접 제어
ctr images list
ctr containers list
ctr task list

# crictl (쿠버네티스 CRI 디버깅)
crictl ps                        # 실행 중 컨테이너
crictl logs <container_id>
crictl exec -it <container_id> sh
crictl inspect <container_id>    # 상세 정보 (pid, cgroup 경로 등)
```

---

## 실무 심화 포인트

- **CPU Throttling 탐지**: `container_cpu_cfs_throttled_seconds_total` (Prometheus). 30% 이상이면 limit 상향 또는 VPA 검토
- **메모리 OOMKill 진단**: `dmesg | grep oom` + `kubectl describe pod` 의 `OOMKilled` 상태 + `container_oom_events_total`
- **Overlay FS와 inotify**: 컨테이너 내 파일 감시(inotify)가 호스트 `fs.inotify.max_user_watches` 제한에 걸릴 수 있음
- **PID 제한**: cgroup `pids.max`로 fork bomb 방어. K8s `spec.containers[].resources.limits.pid` (1.22+)
- **Rootless 컨테이너**: USER namespace로 컨테이너 내 root가 호스트에서 비권한 UID로 매핑. 보안 향상, 일부 기능 제한
- **eBPF로 컨테이너 관찰**: Cilium Hubble — 컨테이너 간 트래픽을 eBPF로 추적, iptables 없이 정책 적용
- **Kata Containers**: VM 기반 컨테이너 격리. namespace 격리 + 가상화 격리 (멀티테넌트 환경)
