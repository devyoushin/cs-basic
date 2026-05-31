# 네트워크 심화 (Advanced Networking)

## 1. BGP (Border Gateway Protocol)

인터넷 전체 라우팅을 담당하는 유일한 EGP(Exterior Gateway Protocol).

### 핵심 개념

```
AS (Autonomous System): 단일 라우팅 정책 하에 운영되는 IP 네트워크 집합
  예: AS15169 (Google), AS7224 (AWS), AS32934 (Meta)

eBGP: 서로 다른 AS 간 라우팅
iBGP: 같은 AS 내부 라우팅
```

### BGP 경로 선택 (Best Path Selection)
```
우선순위 (높을수록 우선):
1. Weight (Cisco 독자 속성, 로컬)
2. LOCAL_PREF (AS 내부 선호도)
3. AS_PATH 길이 (짧을수록 선호)
4. Origin (IGP > EGP > Incomplete)
5. MED (Multi-Exit Discriminator)
6. eBGP > iBGP 경로
7. IGP Metric (다음 홉까지 거리)
```

### BGP 세션 상태 머신
```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

### BGP Route Hijacking
```
문제: 공격자가 타인의 IP 프리픽스를 BGP로 광고
     → 트래픽이 공격자 AS로 라우팅됨
사례: 2008년 파키스탄 텔레콤이 YouTube 전체를 차단
     2010년 중국 차이나텔레콤이 미국 트래픽 15분간 우회

방어:
  - RPKI (Resource Public Key Infrastructure): BGP 경고의 AS-IP 매핑 서명
  - ROV (Route Origin Validation): RPKI로 BGP 경로 검증
  - IRR (Internet Routing Registry): 라우팅 정책 공개 등록
```

### 실무 포인트
- 클라우드 Direct Connect/ExpressRoute는 eBGP로 온프레미스 연결
- Anycast는 동일 IP를 여러 AS에서 광고해 가장 가까운 AS로 라우팅
- BGP Community: 라우팅 정책 태그 (예: no-export, blackhole)

---

## 2. TCP BBR 혼잡 제어 알고리즘

Google이 2016년 개발한 대역폭 기반 혼잡 제어 알고리즘.

### 기존 방식의 문제 (CUBIC/Reno)
```
손실 기반(Loss-based): 패킷 손실이 발생할 때까지 전송량 증가
→ 버퍼 과부하(Bufferbloat): 라우터 버퍼가 꽉 찬 뒤에야 반응
→ 높은 레이턴시 + 낮은 실제 처리량
```

### BBR 동작 원리
```
BtlBw (Bottleneck Bandwidth): 병목 구간 최대 대역폭 추정
RTprop (Round-Trip propagation time): 최소 RTT 측정

전송률 = BtlBw × 이상적 파이프 크기
       ≠ 손실 발생 시점까지 증가

단계:
  STARTUP → 대역폭 포화까지 지수 증가
  DRAIN   → 과도한 큐 비움
  PROBE_BW → BtlBw 주기적 탐색 (8 RTT 사이클)
  PROBE_RTT → RTprop 재측정을 위해 전송량 감소
```

### Linux에서 BBR 활성화
```bash
# BBR 확인
sysctl net.ipv4.tcp_congestion_control

# BBR 활성화
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p

# 확인
sysctl net.ipv4.tcp_congestion_control
# → net.ipv4.tcp_congestion_control = bbr
```

### CUBIC vs BBR 성능
| 구분 | CUBIC | BBR |
|------|-------|-----|
| 기반 | 패킷 손실 | 대역폭 모델 |
| 고속 장거리 링크 | 낮은 처리량 | 2~25배 향상 |
| 레이턴시 | 높음 (큐 적체) | 낮음 |
| 무선/불안정 링크 | 과도한 속도 저하 | 손실 구분 가능 |

---

## 3. Linux conntrack (Connection Tracking)

iptables/nftables의 상태 추적 엔진. 방화벽, NAT, 쿠버네티스 네트워킹의 핵심.

### conntrack 동작
```
모든 패킷은 netfilter를 통과하며 conntrack 테이블에 상태 기록:
  NEW        → SYN 수신, 새 연결 시작
  ESTABLISHED → 양방향 패킷 흐름
  RELATED     → FTP 데이터 연결 등 관련 연결
  INVALID     → 어떤 상태에도 해당하지 않음
```

### conntrack 테이블 확인
```bash
# 현재 추적 중인 연결 수
cat /proc/sys/net/netfilter/nf_conntrack_count

# 최대 허용 연결 수
cat /proc/sys/net/netfilter/nf_conntrack_max

# 연결 목록 확인
conntrack -L

# 특정 IP의 연결
conntrack -L | grep 10.0.0.1

# 실시간 이벤트
conntrack -E
```

### conntrack 테이블 고갈 문제
```
증상: "nf_conntrack: table full, dropping packet" 커널 로그
원인: 대규모 트래픽 처리 서버, UDP 홍수, 과도한 TIME_WAIT

해결:
# 최대 연결 수 증가
echo 1000000 > /proc/sys/net/netfilter/nf_conntrack_max

# 타임아웃 단축
echo 30 > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait
echo 20 > /proc/sys/net/netfilter/nf_conntrack_udp_timeout
```

### 쿠버네티스와 conntrack
- kube-proxy의 iptables 모드는 conntrack에 의존
- 노드당 수천 개 Service → iptables 규칙 폭증 → 성능 저하
- **해결**: ipvs 모드 (해시테이블 기반, O(1) 조회) 또는 eBPF 기반 cilium

---

## 4. VXLAN & Overlay 네트워킹

물리 네트워크(Underlay) 위에 가상 L2 네트워크(Overlay)를 구성.

### VXLAN (Virtual Extensible LAN)
```
특징:
  - L2 이더넷 프레임을 UDP(4789 포트)로 캡슐화
  - VLAN의 4096개 한계 → VNI(VXLAN Network Identifier) 16M개
  - 데이터센터 간 L2 확장 가능

패킷 구조:
  [Outer Ethernet][Outer IP][UDP][VXLAN Header(VNI)][Inner Ethernet][Inner IP][Payload]
```

### VTEP (VXLAN Tunnel Endpoint)
```
VTEP: VXLAN 캡슐화/역캡슐화를 수행하는 장치 (소프트웨어 또는 하드웨어)

패킷 흐름 (VM-A → VM-B, 서로 다른 호스트):
  VM-A → VTEP-1(캡슐화) → 물리 네트워크 → VTEP-2(역캡슐화) → VM-B
```

### 쿠버네티스 CNI에서의 VXLAN
```bash
# Flannel VXLAN 확인
ip -d link show flannel.1    # VXLAN 인터페이스
bridge fdb show dev flannel.1  # VTEP FDB 테이블

# Calico VXLAN
ip -d link show vxlan.calico

# 오버레이 오버헤드: 약 50바이트 → MTU 조정 필요
# 호스트 MTU 1500 → 파드 MTU ~1450
```

### Geneve vs VXLAN
| 구분 | VXLAN | Geneve |
|------|-------|--------|
| 헤더 크기 | 고정 | 가변 (확장 가능) |
| 메타데이터 | 제한적 | 풍부한 TLV |
| 채택 | Flannel, Calico | OVN, Cilium |

---

## 5. iptables / nftables

리눅스 패킷 필터링의 핵심. 방화벽, NAT, 로드밸런싱 구현.

### iptables 구조
```
테이블 (처리 목적):
  filter   → 패킷 허용/차단 (기본)
  nat      → 주소 변환 (SNAT, DNAT)
  mangle   → 패킷 헤더 수정
  raw      → conntrack 이전 처리

체인 (처리 시점):
  PREROUTING  → 라우팅 결정 전 (DNAT 여기서)
  INPUT       → 로컬 프로세스로 향하는 패킷
  FORWARD     → 다른 호스트로 전달되는 패킷
  OUTPUT      → 로컬 프로세스에서 나가는 패킷
  POSTROUTING → 라우팅 결정 후 (SNAT 여기서)
```

### 패킷 흐름
```
수신 패킷:
  → PREROUTING(raw→mangle→nat)
  → 라우팅 결정
     ├─ 로컬: → INPUT(mangle→filter) → 프로세스
     └─ 전달: → FORWARD(mangle→filter) → POSTROUTING → 출력

송신 패킷:
  → OUTPUT(raw→mangle→nat→filter) → POSTROUTING → 출력
```

### 주요 명령어
```bash
# 현재 규칙 확인
iptables -L -n -v --line-numbers
iptables -t nat -L -n -v

# SNAT (아웃바운드 IP 마스킹 - 홈 라우터/NAT GW)
iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE

# DNAT (포트 포워딩 - 인바운드)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:8080

# 특정 IP 차단
iptables -A INPUT -s 1.2.3.4 -j DROP

# 규칙 저장 (재부팅 후 유지)
iptables-save > /etc/iptables/rules.v4
```

### nftables (iptables 후계자)
```bash
# 규칙 확인
nft list ruleset

# 테이블 생성
nft add table inet my_filter
nft add chain inet my_filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet my_filter input tcp dport 22 accept
```

---

## 6. Anycast 라우팅

동일한 IP 주소를 여러 물리적 위치에서 동시에 광고해 가장 가까운 노드로 라우팅.

### 동작 원리
```
전통적 Unicast:
  1.1.1.1 → 특정 서버 1대

Anycast:
  1.1.1.1 → BGP로 여러 POP(Point of Presence)에서 광고
           → 클라이언트는 BGP 경로상 가장 가까운 POP으로 연결
```

### 활용 사례
| 서비스 | Anycast 적용 |
|--------|-------------|
| DNS | 8.8.8.8(Google), 1.1.1.1(Cloudflare), Root Nameserver |
| CDN | Cloudflare, Fastly 전체 네트워크 |
| DDoS 방어 | 트래픽을 여러 POP으로 분산 흡수 |
| NTP | pool.ntp.org |

### Anycast vs 다른 방식
```
Unicast : 1 송신자 → 1 수신자
Broadcast: 1 송신자 → 모든 수신자 (L2 범위)
Multicast: 1 송신자 → 구독 그룹 (IGMP/PIM)
Anycast  : 1 송신자 → 가장 가까운 1 수신자
```

### 실무 제약
- TCP 연결 중 라우팅 변경 시 연결 끊김 (BGP 경로 변동)
- UDP 기반 서비스(DNS, NTP)에 특히 적합
- 장애 시 BGP withdraw로 자동 우회 (수 분 이내)

---

## 7. ECMP (Equal-Cost Multi-Path Routing)

동일 비용의 여러 경로를 동시에 사용해 트래픽 분산.

### 동작 원리
```
목적지 10.0.0.0/24로 가는 동일 비용 경로 3개:
  via 192.168.1.1 (eth0)
  via 192.168.2.1 (eth1)
  via 192.168.3.1 (eth2)

→ 해시 기반으로 각 플로우를 특정 경로에 고정
  해시 키: src IP, dst IP, src Port, dst Port, Protocol
```

### Linux ECMP 설정
```bash
# ECMP 라우팅 추가
ip route add 10.0.0.0/24 \
  nexthop via 192.168.1.1 dev eth0 weight 1 \
  nexthop via 192.168.2.1 dev eth1 weight 1 \
  nexthop via 192.168.3.1 dev eth2 weight 1

# 현재 멀티패스 라우팅 확인
ip route show 10.0.0.0/24
```

### 해시 편향 문제 (Hash Polarization)
```
문제: 특정 플로우가 항상 같은 경로로 몰리는 현상
원인: 여러 계층의 ECMP 장비가 동일한 해시 알고리즘 사용

해결:
  - 해시 시드(seed) 다변화
  - 5-tuple 대신 다른 필드 포함 (VXLAN inner header 등)
```

### 데이터센터 ECMP
```
Spine-Leaf 아키텍처:
  Leaf ↔ Spine 간 여러 링크 → ECMP로 L3 로드밸런싱
  → 특정 링크 장애 시 자동 우회
  → scale-out 시 링크 추가만으로 대역폭 확장
```

---

## 8. DPDK & Kernel Bypass

커널 네트워크 스택을 우회해 패킷을 유저스페이스에서 직접 처리.

### 전통적 패킷 처리 경로의 병목
```
NIC → 드라이버 인터럽트 → 커널 네트워크 스택 → 시스템콜 → 애플리케이션
  ↑ 인터럽트 오버헤드, 컨텍스트 스위칭, 메모리 복사 (최소 2회)
```

### DPDK (Data Plane Development Kit)
```
NIC → DPDK PMD (Poll Mode Driver) → 유저스페이스 애플리케이션

핵심 기술:
  - Hugepage: TLB 미스 감소 (4KB → 2MB/1GB 페이지)
  - Poll Mode: 인터럽트 대신 CPU busy-poll로 레이턴시 최소화
  - Zero-copy: 커널 ↔ 유저스페이스 메모리 복사 제거
  - CPU 고정(affinity): NUMA 로컬 처리

성능: 100Gbps 라인레이트 처리 가능, 서버 1대로 수천만 PPS
적용: OVS-DPDK, VPP (FD.io), SRIOV 가상화
```

### SR-IOV (Single Root I/O Virtualization)
```
물리 NIC 1개를 여러 가상 NIC(VF, Virtual Function)으로 분리:
  PF (Physical Function): 전체 NIC 제어
  VF (Virtual Function): VM/컨테이너에 직접 할당

장점: 하이퍼바이저 소프트웨어 스위칭 제거 → 레이턴시 대폭 감소
적용: AWS ENA Express, Azure Accelerated Networking
```

### XDP (eXpress Data Path)
```
커널 내부에서 패킷 도달 직후 (드라이버 직후) eBPF 프로그램 실행:
  NIC → XDP(eBPF) → 커널 스택 또는 DROP/TX

DPDK과 차이:
  - 커널 우회 X, 커널 내 조기 처리
  - 기존 소켓 API 호환 유지
  - 훨씬 낮은 도입 비용

활용: DDoS 방어(Cloudflare), 로드밸런서(Facebook Katran), cilium
```

---

## 9. QUIC 프로토콜 심화

UDP 기반의 새로운 트랜스포트 레이어 프로토콜 (RFC 9000).

### 핵심 설계 목표
```
문제: TCP는 커널에 구현 → 업데이트 느림, 미들박스(방화벽 등)가 헤더 조작
해결: UDP 위에서 TLS 1.3 통합 → 암호화로 헤더 보호 + 빠른 배포
```

### 주요 특징

**Connection ID 기반 연결 관리**
```
TCP: 4-tuple (src IP, src Port, dst IP, dst Port)로 연결 식별
     → IP 변경(WiFi → LTE) 시 연결 끊김

QUIC: Connection ID로 식별
     → 네트워크 전환(Connection Migration)에도 연결 유지
     → 모바일 환경에 최적
```

**0-RTT 연결 재수립**
```
첫 연결: 1-RTT (TLS 1.3 통합으로 TCP 1.2의 2-RTT에서 단축)
재연결: 0-RTT (이전 세션 티켓 사용)
         → 단, Replay Attack 취약 → GET만 허용 권고
```

**스트림 멀티플렉싱**
```
HTTP/2: TCP 위의 스트림 → 패킷 손실 시 모든 스트림 블로킹 (TCP HOL)
QUIC  : 스트림별 독립 흐름 제어 → 패킷 손실이 다른 스트림에 영향 없음
```

**패킷 번호 단조 증가**
```
TCP: Sequence Number 재사용 → 재전송 모호성(Retransmission Ambiguity)
QUIC: 패킷 번호 단조 증가 → RTT 측정 정확도 향상
```

### QUIC 배포 현황
```bash
# curl로 HTTP/3 테스트
curl --http3 https://cloudflare-quic.com/

# wireshark에서 QUIC 필터
udp.port == 443

# nginx HTTP/3 설정
server {
    listen 443 quic reuseport;
    listen 443 ssl;
    http3 on;
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

---

## 10. 네트워크 네임스페이스 & 가상 인터페이스

컨테이너 네트워크 격리의 핵심 기술.

### 네트워크 네임스페이스
```
리눅스 커널의 격리 기능:
  - 네임스페이스마다 독립적인 네트워크 스택
  - 인터페이스, 라우팅 테이블, iptables 규칙, 소켓 완전 분리

도커/쿠버네티스: 파드/컨테이너마다 별도 netns 생성
```

```bash
# 네임스페이스 생성
ip netns add ns1

# 네임스페이스에서 명령 실행
ip netns exec ns1 ip link show
ip netns exec ns1 bash  # 네임스페이스 내 쉘

# 목록 확인
ip netns list

# 현재 프로세스의 netns 확인
ls -la /proc/self/ns/net
```

### veth (Virtual Ethernet Pair)
```
두 인터페이스가 쌍으로 생성되어 파이프처럼 동작:
  [ns1: eth0] ←→ [호스트: veth-ns1]

컨테이너 네트워킹:
  [파드: eth0] ←→ [호스트: vethXXXXXX] ←→ [cni0 브릿지]
```

```bash
# veth 쌍 생성
ip link add veth0 type veth peer name veth1

# veth1을 ns1으로 이동
ip link set veth1 netns ns1

# 양쪽 IP 설정
ip addr add 192.168.100.1/24 dev veth0
ip netns exec ns1 ip addr add 192.168.100.2/24 dev veth1

ip link set veth0 up
ip netns exec ns1 ip link set veth1 up

# 통신 테스트
ip netns exec ns1 ping 192.168.100.1
```

### 쿠버네티스 파드 네트워크 경로
```
[파드 A: eth0]
     ↕ veth pair
[호스트: vethXXX] → [cni0 브릿지(L2)] → [vethYYY]
                                              ↕ veth pair
                                        [파드 B: eth0]

노드 간:
[파드 A] → [호스트 라우팅] → VXLAN/BGP → [상대 호스트] → [파드 B]
```

### 실무 디버깅
```bash
# 파드의 veth 인터페이스 찾기
# 파드 내부에서 인터페이스 인덱스 확인
kubectl exec -it pod-name -- cat /sys/class/net/eth0/iflink

# 호스트에서 해당 인덱스의 인터페이스 찾기
ip link | grep -A1 "^<인덱스>:"

# 특정 컨테이너 netns 진입 (PID 확인 후)
PID=$(docker inspect --format '{{.State.Pid}}' container-id)
nsenter -t $PID -n ip addr
```

---

## 실무 심화 포인트

- **BGP Graceful Restart**: BGP 세션 재시작 시 인접 라우터가 경로 유지 → 트래픽 단절 방지
- **BBR vs CUBIC 선택**: 장거리/고대역폭 링크, 위성 통신, CDN 엣지 서버에는 BBR이 우월
- **conntrack 튜닝**: 고트래픽 서버에서 `nf_conntrack_max` 부족은 무음 패킷 드롭 원인 → 반드시 모니터링
- **VXLAN MTU 계산**: 호스트 MTU(1500) - VXLAN 오버헤드(50) = 파드 MTU(1450). 잘못 설정 시 대용량 전송에서만 간헐적 장애 발생
- **DPDK 도입 기준**: 일반 서버 수준에서 100Gbps 라인레이트가 필요하거나 수십 마이크로초 단위 레이턴시 요구 시
- **QUIC Alt-Svc 헤더**: HTTP/1.1이나 2로 첫 응답 후 `Alt-Svc: h3=":443"`로 HTTP/3 업그레이드 유도
- **netns 격리 범위**: 네트워크 네임스페이스는 네트워크 스택만 격리. PID, Mount, UTS 등은 별도 네임스페이스
- **Anycast + TCP**: BGP 재수렴 중 TCP 연결 드롭 가능 → Anycast는 DNS/UDP 서비스에, TCP 서비스는 지리 기반 DNS로 분산 권장
- **ECMP 한계**: 코끼리 플로우(대용량 단일 연결)는 분산 불균형 발생 → MPTCP 또는 애플리케이션 레벨 분산 고려
- **XDP vs iptables 성능**: XDP(eBPF) DROP은 iptables 대비 10~100배 빠름 → DDoS 방어에 활용
