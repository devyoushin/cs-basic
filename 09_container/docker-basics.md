# Docker & 컨테이너 기초

## 핵심 멘탈 모델: 3줄 요약

```
Namespace  → 프로세스가 "볼 수 있는 것"을 제한   (격리, 독립된 뷰)
cgroup     → 프로세스가 "쓸 수 있는 양"을 제한   (CPU/메모리 한도)
Kernel     → 모든 컨테이너가 공유                (VM과의 결정적 차이)
```

- **Namespace**: 각 컨테이너에게 "나만의 PID 공간, 나만의 네트워크, 나만의 파일시스템"이라는 착각을 줌
- **cgroup**: 그 착각 속에서도 실제 하드웨어 자원은 공정하게 나눔
- **커널 공유**: Guest OS 부팅이 없어서 수 ms 내 시작 가능. 단, 커널 취약점은 전 컨테이너에 영향

> 한 문장 정의: **컨테이너 = namespace로 격리된 + cgroup으로 제한된 + 커널을 공유하는 프로세스**

---

## 컨테이너의 본질: VM이 아닌 격리된 프로세스

컨테이너는 새로운 OS를 띄우는 게 아님. **호스트 커널을 공유하는 격리된 프로세스**.

```
VM:
┌─────────────────────────────────────┐
│ Guest OS (Linux) + 커널             │
│   └── App                          │
├─────────────────────────────────────┤
│ Hypervisor (VMware, KVM)           │
├─────────────────────────────────────┤
│ Host OS + 커널                      │
└─────────────────────────────────────┘

Container:
┌──────────────┬──────────────┐
│ Container A  │ Container B  │  ← 격리된 프로세스
│   App        │   App        │
│   libc       │   libc       │
└──────────────┴──────────────┘
│       Host Linux Kernel             │  ← 커널 공유
└─────────────────────────────────────┘
```

컨테이너가 가벼운 이유: Guest OS 없음 → 부팅 없음 → 수 ms 만에 시작.
컨테이너의 한계: 커널 공유 → Windows 컨테이너는 Windows 호스트 필요, 커널 취약점은 모든 컨테이너에 영향.

---

## 격리 구현 1: Linux Namespace

Namespace는 프로세스가 **볼 수 있는 자원의 범위**를 제한. 프로세스마다 독립된 뷰 제공.

| Namespace | 격리 대상 | 컨테이너에서의 효과 |
|-----------|---------|----------------|
| **PID** | 프로세스 ID | 컨테이너 내부 PID 1 = 실제 호스트에선 다른 PID |
| **Network** | 네트워크 인터페이스, 라우팅, iptables | 컨테이너별 독립 IP, 포트 |
| **Mount** | 파일시스템 마운트 트리 | 컨테이너별 독립 루트 파일시스템 |
| **UTS** | hostname, domainname | 컨테이너별 독립 hostname |
| **IPC** | System V IPC, POSIX 메시지 큐 | 컨테이너 간 IPC 격리 |
| **User** | UID/GID 매핑 | 컨테이너 내 root = 호스트의 일반 유저 |
| **Cgroup** | cgroup 루트 뷰 | 컨테이너가 자신의 cgroup만 인식 |
| **Time** | 시스템 시간 (Linux 5.6+) | 컨테이너별 독립 시간 |

### PID Namespace 동작
```
Host:
  PID 1 (systemd)
  PID 1234 (dockerd)
  PID 5678 ← 컨테이너의 nginx (호스트 기준)

Container 내부:
  PID 1 ← 같은 nginx 프로세스 (컨테이너 기준)
  PID 2 ← nginx worker
```
- 컨테이너 내 PID 1 프로세스가 종료되면 컨테이너 전체 종료
- `kill -9 1` (컨테이너 내) = 컨테이너 종료 시도 (PID 1은 SIGKILL 무시 가능)

### Network Namespace
```
호스트: eth0 (192.168.1.1)
컨테이너: eth0 (172.17.0.2) ← 가상 인터페이스 (veth pair)

veth pair:
  호스트 쪽: veth3f2a1c → docker0 브리지에 연결
  컨테이너 쪽: eth0 (컨테이너 내)
```

```bash
# 컨테이너의 네트워크 namespace 확인
docker inspect <container> | grep -i pid
ls -la /proc/<PID>/ns/net

# 컨테이너 네트워크 namespace에서 명령 실행
nsenter -t <PID> -n ip addr
```

---

## 격리 구현 2: cgroups (Control Groups)

Namespace가 "볼 수 있는 것"을 제한한다면, cgroups는 **사용할 수 있는 자원의 양**을 제한.

### cgroups v2 계층 구조

```
/sys/fs/cgroup/
├── system.slice/
│   └── docker-<id>.scope/
│       ├── memory.max         ← 메모리 한도
│       ├── memory.current     ← 현재 사용량
│       ├── cpu.max            ← CPU 한도 (quota/period)
│       ├── cpu.stat           ← CPU 통계
│       └── pids.max           ← 최대 프로세스 수
```

### CPU 제한 원리
```
cpu.max = "50000 100000"
         = 100ms 마다 50ms 사용 가능 = 0.5 CPU

docker run --cpus=1.5 → cpu.max = "150000 100000"
```

CFS (Completely Fair Scheduler)의 quota/period 메커니즘:
- `cpu.cfs_period_us` = 측정 주기 (기본 100ms)
- `cpu.cfs_quota_us` = 주기당 허용 CPU 시간
- quota 소진 시 → 스로틀링 (SIGSTOP이 아니라 실행 큐에서 제거)

```bash
# CPU 스로틀링 확인
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/cpu.stat
# throttled_periods: 스로틀된 주기 수
# throttled_time: 스로틀된 총 시간 (나노초)
```

### 메모리 제한 원리
```
memory.max = 512M (하드 한도, 초과 시 OOM Kill)
memory.high = 400M (소프트 한도, 초과 시 GC 압력 증가)
memory.swap.max = 0 (스왑 사용 금지)
```

OOM Kill 우선순위: `oom_score_adj`가 높은 프로세스 먼저 종료.
K8s는 `Guaranteed` 클래스 Pod에 `-997`, `BestEffort`에 `1000` 설정.

---

## 격리 구현 3: Union File System (OverlayFS)

Docker 이미지는 **읽기 전용 레이어의 스택**. 컨테이너는 최상단에 쓰기 가능 레이어 추가.

```
컨테이너 레이어 (읽기-쓰기, 컨테이너 종료 시 삭제)
──────────────────────────────────────────── upperdir
    /app/config.json (수정됨)

Layer 3 (읽기 전용): /app/ 추가               lowerdir
Layer 2 (읽기 전용): /usr/lib/ 추가
Layer 1 (읽기 전용): Base OS (Ubuntu)
──────────────────────────────────────────── lowerdir
```

### OverlayFS 마운트 구조
```bash
mount -t overlay overlay \
  -o lowerdir=/var/lib/docker/overlay2/<id>/diff,\
     upperdir=/var/lib/docker/overlay2/<container-id>/diff,\
     workdir=/var/lib/docker/overlay2/<container-id>/work \
  /var/lib/docker/overlay2/<container-id>/merged
```

- **lowerdir**: 읽기 전용 이미지 레이어들 (콜론으로 구분, 위→아래 우선순위)
- **upperdir**: 컨테이너의 쓰기 레이어
- **workdir**: OverlayFS 내부 임시 작업 디렉토리
- **merged**: 컨테이너가 보는 통합 뷰

### Copy-on-Write (CoW)
파일 수정 시:
1. lowerdir에서 파일 복사 → upperdir로 이동
2. upperdir의 복사본 수정
3. merged에서는 upperdir 파일 우선 노출

→ 이미지 레이어는 절대 수정 안 됨 (여러 컨테이너가 같은 레이어 공유 가능)

```bash
# 레이어 구조 확인
docker history nginx:latest
docker inspect nginx:latest | jq '.[0].RootFS.Layers'

# 컨테이너가 수정한 파일 확인 (upperdir)
docker diff <container>
```

---

## Docker 이미지 구조

### Content-Addressable Storage

```
이미지 = Manifest + Config + 레이어들
각 레이어 = SHA256(tar archive)

nginx:latest
├── manifest.json
│   └── layers: [sha256:abc..., sha256:def..., sha256:ghi...]
├── config.json (CMD, ENV, EXPOSE 등)
└── layers/
    ├── abc.../layer.tar  ← base OS
    ├── def.../layer.tar  ← nginx 바이너리
    └── ghi.../layer.tar  ← nginx 설정
```

SHA256 기반 → 같은 레이어를 여러 이미지가 공유. `ubuntu:20.04`를 기반으로 하는 이미지들은 base 레이어 하나만 저장.

### Dockerfile 레이어 최적화

```dockerfile
# Bad: 레이어마다 캐시 무효화, 불필요한 파일 남김
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl wget
RUN wget https://example.com/app.tar.gz
RUN tar -xzf app.tar.gz
RUN rm app.tar.gz                  # 이미 이전 레이어에 포함됨 → 삭제 무의미

# Good: 하나의 RUN으로 묶기, 임시 파일 같은 레이어에서 삭제
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl wget && \
    wget https://example.com/app.tar.gz && \
    tar -xzf app.tar.gz && \
    rm app.tar.gz && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 캐시 효율: 변경 잦은 레이어를 Dockerfile 하단에
COPY requirements.txt .        # 의존성 (덜 변경)
RUN pip install -r requirements.txt
COPY . .                       # 소스코드 (자주 변경) ← 마지막에
```

### Multi-stage Build
```dockerfile
# Stage 1: 빌드 환경 (최종 이미지에 포함 안 됨)
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

# Stage 2: 실행 환경 (최소화)
FROM scratch                    # 빈 이미지 (또는 alpine, distroless)
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/server"]
# 결과: golang 빌드 도구 없는 수 MB짜리 이미지
```

---

## 컨테이너 런타임 스택

```
Docker CLI
    ↓
dockerd (Docker daemon)
    ↓
containerd (CNCF 표준 런타임)
    ↓
containerd-shim
    ↓
runc (OCI 런타임 스펙 구현)
    ↓
Linux Kernel (namespace, cgroups, OverlayFS)
```

- **OCI (Open Container Initiative)**: 컨테이너 이미지/런타임 표준 규격
- **containerd**: Docker, K8s 모두 사용하는 업계 표준 컨테이너 런타임
- **runc**: namespace/cgroup 설정 후 프로세스 실행. Go + C 구현
- **gVisor (runsc)**: runc 대신 사용. 사용자 공간 커널로 syscall 인터셉트 → 강한 격리

---

## Docker 네트워킹 심화

### OSI 계층 관점: 브리지 vs 라우터

```
OSI 계층    장비/역할              동작 방식
─────────────────────────────────────────────────────
L2 (Data Link)  스위치 / 브리지    MAC 주소로 같은 네트워크 내 전달
L3 (Network)    라우터             IP 주소로 다른 네트워크 간 경로 결정
L4 (Transport)  방화벽 / LB        TCP/UDP 포트로 필터링·분산
L7 (Application) Nginx / ALB       HTTP URL·헤더로 라우팅
```

**브리지 (docker0)**:
- L2 장비. 같은 브리지에 연결된 컨테이너끼리는 MAC 주소 기반으로 직접 통신
- 컨테이너 A → 컨테이너 B: IP 패킷이 docker0에서 L2 전달 (라우팅 불필요)

**라우터 역할 (호스트 커널)**:
- 컨테이너 → 외부 인터넷: 호스트 커널이 L3 라우터처럼 동작
- `ip_forward=1` 설정으로 패킷을 docker0 → eth0으로 포워딩
- iptables MASQUERADE로 컨테이너 IP → 호스트 IP로 SNAT

```
컨테이너 A (172.17.0.2) ──L2 veth──→ docker0 (L2 브리지)
                                           │
                                    커널 L3 포워딩 (라우터 역할)
                                           │
                                    iptables MASQUERADE (SNAT)
                                           │
                                        eth0 → Internet
```

---

### Bridge 네트워크 (기본)

```
Host
├── eth0 (192.168.1.100) ← 외부 인터페이스
└── docker0 (172.17.0.1) ← 브리지 (L2 스위치 역할)
    ├── veth3a1b2c → Container A (172.17.0.2)
    └── veth8f3d4e → Container B (172.17.0.3)
```

컨테이너 → 외부 통신 흐름:
```
Container A (172.17.0.2) → veth → docker0 → MASQUERADE (SNAT) → eth0 → Internet
```

iptables 규칙 (Docker 자동 생성):
```bash
# MASQUERADE: 컨테이너 IP → 호스트 IP로 변환
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

# 포트 포워딩: 호스트:8080 → 컨테이너:80
iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

### 네트워크 모드 비교

| 모드 | 설명 | 격리 | 성능 |
|------|------|------|------|
| bridge | 기본, NAT 통해 외부 통신 | O | 보통 |
| host | 호스트 네트워크 직접 사용 | X | 최고 |
| none | 네트워크 없음 | O | - |
| overlay | 멀티 호스트 (Swarm/K8s) | O | NAT 없음 |

```bash
# host 모드: 포트 포워딩 없이 직접 바인딩 (성능 최대, 격리 없음)
docker run --network=host nginx
```

---

## 컨테이너 보안 심화

### Capability (권한 세분화)
Linux root 권한을 기능별로 분리. 컨테이너는 기본적으로 일부만 허용.

```bash
# 기본 허용: CHOWN, DAC_OVERRIDE, FOWNER, NET_BIND_SERVICE, SETUID...
# 기본 제거: SYS_ADMIN, NET_ADMIN, SYS_PTRACE...

# 최소 권한 (모두 제거 후 필요한 것만)
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

### Seccomp 프로파일
허용할 syscall 화이트리스트:
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {"names": ["read", "write", "open", "close", "stat"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["socket", "connect", "accept"], "action": "SCMP_ACT_ALLOW"}
  ]
}
```
- Docker 기본 Seccomp 프로파일: 약 300개 syscall 중 ~44개 차단
- `--security-opt seccomp=unconfined`: 비활성화 (권장 안 함)

---

## 실무 포인트

- **컨테이너 메모리 = Xmx + Native** (Java): `-Xmx`를 컨테이너 제한의 75% 이하로. 나머지는 Metaspace, 스레드 스택, Direct Buffer
- **OOM Kill 원인**: `dmesg | grep -i "killed process"`. cgroup 한도 초과 시 발생. `memory.stat`에서 `oom_kill` 카운터 확인
- **레이어 캐시 활용**: `COPY requirements.txt` → `RUN pip install` 순서. 소스 변경 시 pip install 재실행 방지
- **read-only 컨테이너**: `docker run --read-only` + tmpfs 마운트. 런타임 파일 시스템 변경 불가 → 침해 시 영속 불가
- **`docker stats`**: CPU/메모리 실시간 확인. `NET I/O`, `BLOCK I/O` 모니터링
- **cgroup v2 전환**: Ubuntu 21.10+, RHEL 9+ 기본. K8s 1.25+에서 정식 지원. `memory.oom.group`으로 Pod 전체 OOM Kill 가능
