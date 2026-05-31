# OSI 7계층 & TCP/IP

## OSI 7계층

| 계층 | 이름 | 프로토콜/기술 | DevOps 관련 |
|------|------|-------------|------------|
| 7 | Application | HTTP, DNS, SSH, SMTP | API, 웹서버 |
| 6 | Presentation | TLS/SSL, JPEG | 암호화 |
| 5 | Session | RPC, NetBIOS | 세션 관리 |
| 4 | Transport | TCP, UDP | 포트, 방화벽 |
| 3 | Network | IP, ICMP, BGP | 라우팅, L3 스위치 |
| 2 | Data Link | Ethernet, MAC, ARP | L2 스위치, VLAN |
| 1 | Physical | 케이블, 광섬유 | 물리 인프라 |

실무에서는 주로 **L3(IP), L4(TCP/UDP), L7(HTTP)** 위주로 다룸.

---

## TCP 3-Way Handshake

```
Client                    Server
  |  ── SYN(seq=x) ──→  |   LISTEN
  |  ←─ SYN+ACK ─────  |   SYN_RCVD
  |  ── ACK ──────────→ |   ESTABLISHED
  |                      |   ESTABLISHED
```

- **SYN**: 연결 시작 요청
- **SYN+ACK**: 연결 수락
- **ACK**: 확인

### TCP 4-Way Handshake (연결 종료)
```
Client                    Server
  |  ── FIN ──────────→ |
  |  ←─ ACK ─────────  |
  |  ←─ FIN ─────────  |
  |  ── ACK ──────────→ |
  |        [TIME_WAIT]  |
```

**TIME_WAIT** 상태: 마지막 ACK가 유실될 경우를 대비해 2MSL(보통 60초) 대기.
대량의 TIME_WAIT = 연결이 빠르게 생성/소멸됨 (단기 HTTP 요청). 보통 정상.

---

## TCP vs UDP

| 구분 | TCP | UDP |
|------|-----|-----|
| 연결 | 연결 지향 (handshake) | 비연결 |
| 신뢰성 | 보장 (재전송) | 없음 |
| 순서 | 보장 | 없음 |
| 속도 | 느림 | 빠름 |
| 오버헤드 | 큼 | 작음 |
| 사용 | HTTP, SSH, DB | DNS, DHCP, 영상스트리밍, 게임 |

---

## TCP 흐름 제어 & 혼잡 제어

### 슬라이딩 윈도우 (흐름 제어)
- 수신자가 처리할 수 있는 양만큼 전송 조절
- `rwnd` (Receive Window): 수신 버퍼 여유 공간
- 수신 버퍼가 가득 차면 `rwnd=0` 전송 → 송신 중단 → 버퍼 여유 생기면 재개

### 혼잡 제어 (Congestion Control)

**CWND (Congestion Window)**: 수신자 허용(rwnd)과 별도로 네트워크 혼잡 기반 송신량 제한.
실제 전송량 = min(rwnd, cwnd)

```
cwnd
  │           /
  │          /          ← Congestion Avoidance (+1/RTT)
  │         /
ssthresh──/─────────────────
  │      /                  → 3 dup ACK: ssthresh/2, cwnd→ssthresh (Fast Recovery)
  │    /                    → Timeout: ssthresh/2, cwnd=1 (Slow Start 재시작)
  │  / ← Slow Start (×2/RTT)
  │/
  └──────────────────────── time
```

| 단계 | 동작 |
|------|------|
| Slow Start | cwnd=1 MSS에서 시작. ACK마다 cwnd×2 (지수 증가). ssthresh 도달 시 전환 |
| Congestion Avoidance | cwnd를 RTT당 1 MSS씩 선형 증가 |
| Fast Retransmit | 3 duplicate ACK → 타임아웃 기다리지 않고 즉시 재전송 |
| Fast Recovery | Fast Retransmit 후 ssthresh=cwnd/2, cwnd=ssthresh → Congestion Avoidance 재개 |

### TCP Nagle 알고리즘
- 작은 패킷을 모아 전송 (ACK 미수신 상태에서 전송 대기)
- 대용량 전송에 유리, 인터랙티브 서비스에 불리 (`TCP_NODELAY`로 비활성화)

---

## IP 주소 & 서브넷

```
IP 주소: 192.168.1.100/24
          ↑            ↑
          호스트 주소   서브넷 마스크 (CIDR)

/24 = 255.255.255.0
  → 네트워크: 192.168.1.0
  → 브로드캐스트: 192.168.1.255
  → 사용 가능 호스트: 192.168.1.1 ~ .254 (254개)
```

### 사설 IP 대역
```
10.0.0.0/8         (10.x.x.x)
172.16.0.0/12      (172.16.x.x ~ 172.31.x.x)
192.168.0.0/16     (192.168.x.x)
```

### 유용한 CIDR 계산
| CIDR | 호스트 수 | 용도 |
|------|---------|------|
| /32 | 1 | 단일 호스트 |
| /30 | 4 (2 사용) | P2P 링크 |
| /28 | 16 (14 사용) | 소규모 서브넷 |
| /24 | 256 (254 사용) | 일반 서브넷 |
| /16 | 65,536 | 대규모 VPC |

---

## 주요 포트

| 포트 | 서비스 |
|------|--------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |
| 2181 | Zookeeper |
| 9092 | Kafka |
| 2379/2380 | etcd |

---

## 실무 포인트

- **ss -s** = 소켓 통계 요약. ESTABLISHED, TIME_WAIT 수 빠르게 파악
- **CLOSE_WAIT 증가** = 서버가 FIN을 보내지 않음 → 애플리케이션 버그 의심
- **SYN Flood 공격** = 대량의 SYN만 보내고 ACK 안 함 → SYN Cookies로 방어
- **TCP Keep-Alive**: 유휴 연결 확인 (`/proc/sys/net/ipv4/tcp_keepalive_time`)
- **MTU (1500 bytes)**: 이보다 큰 패킷은 단편화(Fragmentation). VPN/터널링에서 MTU 조정 필요
- **Jumbo Frame (9000 bytes)**: 데이터센터 내부 고성능 네트워크에서 사용
