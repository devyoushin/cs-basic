# TCP 내부 동작 & 커널 네트워크 튜닝 (Senior Level)

## TCP 혼잡 제어 알고리즘 심화

### 혼잡 감지 방법
- **Tahoe/Reno**: 패킷 손실(timeout 또는 3 duplicate ACK)을 혼잡 신호로 사용
- **CUBIC** (Linux 기본): 손실 기반. cubic 함수로 window 증가
- **BBR** (Google): 손실 대신 실측 대역폭(bandwidth)과 RTT를 모델링

### CUBIC vs BBR
```
CUBIC:
- 손실 발생까지 window 증가 → 버퍼 가득 채움 → 손실 → 감소 반복
- Bufferbloat 문제: 버퍼에 패킷이 가득 차서 레이턴시 급증

BBR (Bottleneck Bandwidth and Round-trip propagation time):
- "최대 처리량을 유지하면서 버퍼를 최소화"
- RTprop(최소 RTT) + BtlBw(최대 대역폭) 지속 측정
- 손실 없이도 전송률 조절
- 패킷 손실이 많은 WAN, 위성통신에서 CUBIC 대비 월등
```

```bash
# 현재 혼잡 제어 알고리즘 확인
sysctl net.ipv4.tcp_congestion_control

# 사용 가능한 알고리즘 목록
sysctl net.ipv4.tcp_available_congestion_control

# BBR 활성화
modprobe tcp_bbr
sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl -w net.core.default_qdisc=fq  # BBR과 함께 FQ 스케줄러 필수

# 연결별 혼잡 제어 알고리즘 확인
ss -tin | grep -A1 "ESTAB" | grep cc
```

### TCP 슬로우 스타트 & CWND
```
Initial CWND (IW): 10 segments (RFC 6928)
Slow Start: RTT마다 CWND 2배 증가 (ssthresh까지)
Congestion Avoidance: ssthresh 이후 RTT마다 1 MSS 증가
```

```bash
# CWND 실시간 확인
ss -tin dst <server_ip> | grep cwnd

# Google's IW10 (InitRTO) 설정 - 초기 연결 빠르게
ip route change default via <gw> initcwnd 10 initrwnd 10
```

---

## TCP 버퍼 & 소켓 튜닝

```bash
# 소켓 버퍼 크기 (BDP 계산)
# BDP = Bandwidth × RTT
# 1Gbps * 100ms RTT = 125MB/s * 0.1s = 12.5MB

# 자동 튜닝 활성화 (기본 on)
sysctl net.ipv4.tcp_moderate_rcvbuf   # 1 = 활성화

# 최대 버퍼 크기
sysctl net.core.rmem_max              # 수신 최대 (기본 212992 = 208KB, 너무 작음)
sysctl net.core.wmem_max              # 송신 최대

# TCP별 버퍼: min / default / max
sysctl net.ipv4.tcp_rmem              # "4096 131072 6291456"
sysctl net.ipv4.tcp_wmem

# 고대역폭 네트워크 최적화 (10G+ NIC, 고레이턴시 WAN)
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"
```

---

## TIME_WAIT 대규모 처리

```bash
# TIME_WAIT 상태 소켓 수 확인
ss -s | grep TIME-WAIT
cat /proc/net/sockstat | grep TCP

# TIME_WAIT 소켓 재사용 (클라이언트)
sysctl net.ipv4.tcp_tw_reuse=1     # TIME_WAIT 소켓 재사용 (안전)
# tcp_tw_recycle은 제거됨 (NAT 환경에서 패킷 드롭 문제)

# 로컬 포트 범위 확장 (기본 32768-60999 = 28231개)
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# SO_REUSEADDR / SO_REUSEPORT
# SO_REUSEPORT: 여러 소켓이 같은 포트 바인딩 → 멀티코어 분산
```

### Ephemeral Port 고갈
```bash
# 포트 고갈 증상
ss -s
# TIME-WAIT: 28000+ → 포트 풀 고갈 임박

# 해결:
# 1. Keep-Alive 활성화로 연결 재사용
sysctl net.ipv4.tcp_keepalive_time=60
sysctl net.ipv4.tcp_keepalive_intvl=10
sysctl net.ipv4.tcp_keepalive_probes=6

# 2. Connection Pool 사이즈 조정 (앱 레벨)
# 3. 포트 범위 확장
# 4. 멀티 IP로 포트 풀 확장 (IP Aliasing)
```

---

## TCP 연결 큐 (Backlog)

```
SYN 수신
  → SYN Queue (Incomplete 연결, SYN_RCVD 상태)
    → 3-way handshake 완료
      → Accept Queue (Complete 연결, ESTABLISHED 상태)
        → app이 accept() 호출
```

```bash
# SYN Queue 크기
sysctl net.ipv4.tcp_max_syn_backlog   # 기본 1024

# Accept Queue 크기 (listen() backlog 인수와 이 값 중 작은 값)
sysctl net.core.somaxconn             # 기본 4096

# 큐 오버플로 확인
netstat -s | grep -i "listen\|overflow\|drop"
# "X SYNs to LISTEN sockets dropped" 증가 → SYN 큐 부족

# nginx listen backlog 설정
listen 80 backlog=65535;

# SYN Cookies (SYN 큐 없이 SYN Flood 방어)
sysctl net.ipv4.tcp_syncookies=1
```

---

## TCP Fast Open (TFO)

3-way handshake 없이 첫 SYN에 데이터 포함:
```
일반: SYN → SYN+ACK → ACK → Data (1 RTT 추가)
TFO:  SYN+Data → SYN+ACK+Data (첫 연결 이후)
```

```bash
# TFO 활성화 (서버=1, 클라이언트=2, 양쪽=3)
sysctl -w net.ipv4.tcp_fastopen=3

# nginx TFO
listen 80 fastopen=256 reuseport;
```

---

## Netfilter / iptables 내부

```
패킷 수신 경로의 훅 포인트:
PREROUTING → FORWARD → POSTROUTING
     ↓
  INPUT (로컬 프로세스)

체인 처리 순서:
  iptables tables: raw → mangle → nat → filter
```

### nftables (iptables 대체)
```bash
# 현재 규칙 확인
nft list ruleset

# 연결 추적 (conntrack) 테이블
conntrack -L | wc -l           # 현재 추적 중인 연결 수
sysctl net.netfilter.nf_conntrack_max   # 최대 추적 가능 연결 수
# conntrack 테이블 가득 차면 새 연결 드롭 → 심각한 장애

# conntrack 튜닝 (대규모 서버)
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600
```

---

## 고성능 네트워크 패턴

### RSS / RPS / RFS
```
RSS (Receive Side Scaling): NIC 하드웨어가 패킷을 여러 RX 큐로 분산
RPS (Receive Packet Steering): 소프트웨어 RSS (단일 큐 NIC용)
RFS (Receive Flow Steering): 패킷을 해당 소켓을 처리하는 CPU로 전달
```

```bash
# NIC 큐 수 확인 및 조정
ethtool -l eth0
ethtool -L eth0 combined 8     # 8개 큐 (CPU 코어 수에 맞춤)

# IRQ를 각 CPU에 분산
cat /proc/interrupts | grep eth0
echo 1 > /proc/irq/<N>/smp_affinity  # CPU 0에 IRQ 고정
```

### 커널 바이패스 (DPDK)
```
일반: NIC → 커널 스택 → User Space (syscall 오버헤드)
DPDK: NIC → User Space (폴링 방식, 제로 카피)

사용 사례: OVS-DPDK (가상화 네트워크), 고성능 패킷 처리
처리량: 수천만 PPS 가능
```

---

## 실무 심화 포인트

- **TCP Retransmission 분석**: `tcpdump`로 캡처 후 Wireshark의 "Expert Info" → 재전송 패턴으로 혼잡/링크 오류 구분
- **RTT 측정 정확도**: `ss -tin`의 `rtt` 값이 앱 레이턴시와 크게 다르면 앱 처리 지연 의심
- **Nagle 알고리즘**: 작은 패킷을 모아서 전송. 인터랙티브 앱(SSH, telnet)에서 `TCP_NODELAY` 소켓 옵션으로 비활성화
- **Receive Buffer Autotuning이 동작 안 하는 경우**: 앱이 `SO_RCVBUF`를 명시적으로 설정하면 autotuning 비활성화됨
- **TCP_CORK**: HTTP 응답 헤더+바디를 한 번에 전송할 때 sendfile+TCP_CORK 조합으로 효율 향상
- **`/proc/net/tcp`**: 모든 TCP 소켓 상태를 16진수로 덤프. `ss`가 없는 환경에서 파싱 가능
