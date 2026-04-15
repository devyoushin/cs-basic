# iptables & NAT

## Netfilter: 리눅스 커널 패킷 처리 프레임워크

iptables는 커널의 **Netfilter** 프레임워크를 제어하는 사용자 공간 도구.
패킷이 커널을 통과하는 5개의 훅 포인트에 규칙을 삽입.

```
네트워크 인터페이스 수신
         ↓
    PREROUTING     ← NAT (DNAT), 라우팅 전 처리
         ↓
    라우팅 결정 (로컬 프로세스 vs 포워딩?)
    ↙               ↘
INPUT            FORWARD         ← 다른 호스트로 포워딩
  ↓                  ↓
로컬 프로세스     POSTROUTING     ← NAT (SNAT/MASQUERADE)
  ↓                  ↓
OUTPUT           네트워크 인터페이스 송신
  ↓
POSTROUTING
  ↓
네트워크 인터페이스 송신
```

---

## 테이블 (Table): 목적별 규칙 그룹

| 테이블 | 목적 | 사용 가능 체인 |
|--------|------|-------------|
| **filter** | 패킷 허용/차단 (기본) | INPUT, FORWARD, OUTPUT |
| **nat** | 주소 변환 (NAT) | PREROUTING, POSTROUTING, OUTPUT |
| **mangle** | 패킷 헤더 수정 (TTL, TOS, Mark) | 모든 체인 |
| **raw** | 연결 추적 제외 (성능) | PREROUTING, OUTPUT |
| **security** | SELinux/AppArmor 연계 | INPUT, FORWARD, OUTPUT |

---

## 규칙 구조

```bash
iptables -t <table> -A <chain> <match> -j <target>

# 예시
iptables -t filter -A INPUT \
  -p tcp \                        # 프로토콜
  --dport 22 \                    # 목적지 포트
  -s 203.0.113.0/24 \             # 출발지 IP
  -m state --state NEW,ESTABLISHED \  # 연결 상태
  -j ACCEPT                       # 액션
```

### Target (액션)
| Target | 설명 |
|--------|------|
| ACCEPT | 패킷 허용 |
| DROP | 패킷 무시 (응답 없음) |
| REJECT | 패킷 거부 (응답 전송: ICMP 또는 RST) |
| LOG | 로그 기록 후 다음 규칙 계속 |
| DNAT | 목적지 IP/포트 변환 (PREROUTING) |
| SNAT | 출발지 IP 변환 (POSTROUTING) |
| MASQUERADE | SNAT의 동적 버전 (IP가 동적으로 바뀔 때) |
| RETURN | 현재 체인에서 상위 체인으로 반환 |

---

## 연결 추적 (Connection Tracking, conntrack)

iptables가 Stateful 방화벽으로 동작하게 하는 핵심.

```
패킷 수신 → conntrack 테이블 조회
  ┌── 신규 연결 (NEW): 새 항목 생성
  ├── 기존 연결 (ESTABLISHED): 이미 허용된 연결
  ├── 관련 연결 (RELATED): FTP data channel, ICMP 오류 등
  └── 유효하지 않음 (INVALID): DROP 권장
```

```bash
# conntrack 테이블 조회
conntrack -L                          # 모든 추적 연결
conntrack -L | grep "dport=443"       # 특정 포트 연결
cat /proc/net/nf_conntrack            # 커널 파일

# 현재 추적 중인 연결 수 / 최대값
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max  # 기본 65536

# conntrack 테이블 고갈 증상: dmesg에 "nf_conntrack: table full, dropping packet"
# 해결:
sysctl -w net.netfilter.nf_conntrack_max=262144
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600  # 기본 5일 → 10분
```

---

## NAT (Network Address Translation)

### SNAT (Source NAT): 출발지 주소 변환

```
Private Network              Internet
10.0.0.2:54321 ─────────→ [SNAT] ─────────→ 1.1.1.1:443
                          203.0.113.1:54321
                          (출발지 IP 변환)
```

```bash
# SNAT: 고정 IP로 변환 (출발지 IP가 고정일 때)
iptables -t nat -A POSTROUTING \
  -s 10.0.0.0/8 \
  -o eth0 \
  -j SNAT --to-source 203.0.113.1

# MASQUERADE: 동적 IP로 변환 (출발지 IP가 동적일 때, 클라우드/DHCP 환경)
iptables -t nat -A POSTROUTING \
  -s 10.0.0.0/8 \
  ! -d 10.0.0.0/8 \
  -o eth0 \
  -j MASQUERADE
```

MASQUERADE는 인터페이스의 현재 IP를 자동으로 읽어 사용. IP 변경 시 자동 적용.
단, MASQUERADE는 SNAT보다 약간 느림 (매 패킷마다 IP 조회).

### DNAT (Destination NAT): 목적지 주소 변환 (포트 포워딩)

```
외부 클라이언트 → 203.0.113.1:80 ─→ [DNAT] ─→ 10.0.0.2:8080 (내부 서버)
```

```bash
# 포트 포워딩: 호스트 80 → 내부 서버 10.0.0.2:8080
iptables -t nat -A PREROUTING \
  -p tcp \
  --dport 80 \
  -j DNAT --to-destination 10.0.0.2:8080

# 위 DNAT가 작동하려면 FORWARD도 허용해야 함
iptables -t filter -A FORWARD \
  -p tcp \
  -d 10.0.0.2 \
  --dport 8080 \
  -j ACCEPT

# IP 포워딩 활성화 (라우터 역할 허용)
echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl -w net.ipv4.ip_forward=1
```

---

## 실용 방화벽 설정 패턴

### 기본 보안 정책 (Default Deny)

```bash
# 기본 정책: 모두 DROP
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT   # 아웃바운드는 허용 (변경 가능)

# Loopback 허용
iptables -A INPUT -i lo -j ACCEPT

# 기존 연결 유지
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH 허용 (특정 IP만)
iptables -A INPUT -p tcp --dport 22 -s 203.0.113.0/24 -j ACCEPT

# 웹 서비스 허용
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# ICMP 허용 (ping)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
```

### 규칙 저장/복원
```bash
# 저장
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

# 복원
iptables-restore < /etc/iptables/rules.v4

# systemd (iptables-persistent 패키지)
systemctl enable netfilter-persistent
```

---

## Docker/K8s에서 iptables 역할

### Docker 브리지 네트워킹

```bash
# Docker가 자동 생성하는 iptables 규칙
iptables -t nat -L DOCKER       # 포트 포워딩 규칙
iptables -t filter -L DOCKER    # 컨테이너 간 통신 규칙

# docker run -p 8080:80 nginx 실행 시 자동 생성:
-A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
-A FORWARD -d 172.17.0.2/32 -p tcp --dport 80 -j ACCEPT
```

### K8s Service (ClusterIP) 구현

```bash
# kube-proxy가 생성하는 규칙 (Service 10.96.100.50:80 → Pod들)
iptables -t nat -L KUBE-SERVICES | grep 10.96.100.50

# 실제 로드밸런싱 체인
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.33333 -j KUBE-SEP-AAA
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.50000 -j KUBE-SEP-BBB
-A KUBE-SVC-XXXX -j KUBE-SEP-CCC
```

### K8s NetworkPolicy 구현 (Calico 예시)
```bash
# Pod 간 통신 차단 체인
iptables -L cali-INPUT   # Calico가 생성한 체인
# 특정 CIDR/포트만 허용하는 규칙이 삽입됨
```

---

## nftables (iptables의 후계자)

Linux 3.13+. 더 나은 성능과 유연성. RHEL 8+, Ubuntu 20.04+에서 기본.

```bash
# nftables 기본 구조 (테이블 → 체인 → 규칙)
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept

# 현재 규칙 확인
nft list ruleset

# iptables → nftables 변환 도구
iptables-translate -A INPUT -p tcp --dport 80 -j ACCEPT
# → nft add rule ip filter INPUT tcp dport 80 accept
```

### nftables 성능 장점
- 여러 매칭을 **Set/Map**으로 O(1) 처리 (iptables는 O(n) 선형 스캔)
- 원자적 규칙 교체 (iptables는 규칙별 락)
- 단일 커맨드로 IPv4/IPv6 동시 처리

---

## 실무 포인트

- **conntrack 테이블 고갈**: 고트래픽 서버에서 흔한 문제. `nf_conntrack_max` 늘리고 `established` 타임아웃 줄이기 (기본 5일은 너무 길다)
- **iptables 규칙 순서**: 위에서 아래로 첫 매칭에서 처리. 자주 매칭되는 규칙을 위에 배치 (성능)
- **Docker와 iptables 충돌**: `ufw` + Docker 사용 시 Docker가 UFW를 우회해 포트를 직접 열 수 있음. `iptables: false` 또는 `127.0.0.1:8080:80` 바인딩으로 제한
- **K8s kube-proxy iptables 모드 한계**: Service 1000개 이상이면 규칙 수만 개 → 패킷마다 선형 스캔. IPVS 모드 또는 Cilium(eBPF)로 전환 권장
- **FORWARD 체인과 ip_forward**: 컨테이너/VM 라우팅을 위해 `ip_forward=1` 설정. 미설정 시 패킷이 목적지에 도달 안 됨
- **방화벽 변경은 항상 테스트**: SSH 원격 접속 중에 INPUT 기본 정책을 DROP으로 바꾸면 접속 끊김. 반드시 `at` 명령어로 자동 복구 예약
  ```bash
  echo "iptables -F; iptables -P INPUT ACCEPT" | at now + 5 minutes
  iptables -P INPUT DROP   # 5분 후 자동 복구
  ```
