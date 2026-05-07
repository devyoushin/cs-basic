# TCP 완전 분석 — 커널 내부부터 VPC Flow Log까지

> SRE/DevOps Senior 레벨: 패킷 한 장이 어떻게 전달되는지 커널 코드 흐름부터 클라우드 관측까지

---

## 1. TCP 상태 머신 (State Machine)

TCP는 11개 상태로 구성된 유한 상태 기계(FSM)다.

```
                              ┌─────────────────────────────────────┐
                              ▼                                     │
[CLOSED] ──listen()──→ [LISTEN]                                     │
    │                      │                                        │
connect()              SYN 수신                                     │
    │                      │                                        │
    ▼                      ▼                                        │
[SYN_SENT] ←────── [SYN_RCVD] ──────────────────────────────────→ [ESTABLISHED]
    │          SYN+ACK          ACK 수신                             │
    │ SYN+ACK 수신                                              close() or
    └──────────────────────────────────────────────────────────  FIN 수신
                                                                     │
              ┌──────────────────────────────────────────────────────┤
              │ 능동 종료(Active Close)      │ 수동 종료(Passive Close)│
              ▼                             ▼                        │
        [FIN_WAIT_1]                  [CLOSE_WAIT] ←─────────────── FIN 수신
              │                             │
           ACK 수신                    close() 호출
              │                             │
              ▼                             ▼
        [FIN_WAIT_2]                   [LAST_ACK]
              │                             │
         FIN 수신                       ACK 수신
              │                             │
              ▼                             ▼
         [TIME_WAIT]                    [CLOSED]
              │
         2MSL 경과
              │
              ▼
          [CLOSED]
```

### 각 상태별 커널 동작

| 상태 | 설명 | 실무 의미 |
|------|------|-----------|
| LISTEN | `accept()` 대기 중. SYN Queue, Accept Queue 준비 | `ss -lnt`로 확인 |
| SYN_SENT | SYN 전송 후 SYN+ACK 대기. `tcp_syn_retries` 횟수만큼 재시도 | 서버 응답 없음 = 방화벽/서버 다운 |
| SYN_RCVD | SYN Queue에 있는 상태. 3-way handshake 미완료 | SYN Flood 시 이 상태 폭증 |
| ESTABLISHED | 데이터 전송 가능 상태 | 정상 연결. 다수면 부하 높음 |
| FIN_WAIT_1 | FIN 전송 후 ACK 대기 | 짧게 지나감, 관찰 어려움 |
| FIN_WAIT_2 | ACK 수신, 서버 FIN 대기 | 서버가 FIN을 안 보내면 고착 |
| CLOSE_WAIT | 클라이언트 FIN 수신, 앱이 close() 안 함 | **버그 신호**: 앱이 소켓을 닫지 않음 |
| LAST_ACK | FIN 전송 후 마지막 ACK 대기 | 금방 사라짐 |
| TIME_WAIT | 연결 종료 후 2MSL 대기 | 대량이어도 보통 정상 |
| CLOSING | 양쪽이 동시에 FIN 전송 (드문 케이스) | 거의 볼 일 없음 |
| CLOSED | 연결 없음 | |

```bash
# 상태별 소켓 수 한눈에 보기
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# 특정 포트의 소켓 상태
ss -tan sport = :443

# TIME_WAIT 상태만
ss -tan state time-wait | head -20

# CLOSE_WAIT 상태만 (버그 탐지)
ss -tan state close-wait
```

---

## 2. TCP 헤더 구조 심화

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌─────────────────────────┬─────────────────────────────────────┐
│      Source Port        │       Destination Port              │  ← 각 16bit
├─────────────────────────────────────────────────────────────────┤
│                    Sequence Number                              │  ← 32bit
├─────────────────────────────────────────────────────────────────┤
│                 Acknowledgment Number                           │  ← 32bit
├──────────┬──────────────┬─────────────────────────────────────┤
│Data Offset│  Reserved   │ C E U A P R S F  (Control Bits)     │
│  (4bit)   │   (3bit)   │ W C R C S S Y I                     │
│           │             │ R E G K H T N N                     │
├──────────┴──────────────┴─────────────────────────────────────┤
│         Window Size     │          Checksum                    │
├─────────────────────────┬───────────────────────────────────────┤
│      Urgent Pointer     │          Options (가변 길이)           │
└─────────────────────────┴───────────────────────────────────────┘
```

### 제어 비트 (Flags) 상세

| 비트 | 이름 | 역할 |
|------|------|------|
| SYN | Synchronize | 연결 시작. ISN(Initial Sequence Number) 동기화 |
| ACK | Acknowledgment | ACK Number 유효함 표시. SYN 이후 항상 set |
| FIN | Finish | 연결 종료 요청 |
| RST | Reset | 연결 즉시 강제 종료 (비정상 상황) |
| PSH | Push | 버퍼 비우지 말고 즉시 애플리케이션에 전달 |
| URG | Urgent | 긴급 데이터 (거의 안 씀) |
| ECE | ECN Echo | 혼잡 신호 수신 (ECN 사용 시) |
| CWR | Congestion Window Reduced | 혼잡 제어 응답 완료 |

### TCP Options (핵심 4가지)

```
MSS (Maximum Segment Size):
  - SYN 패킷에서만 협상
  - 기본값: MTU - IP헤더(20) - TCP헤더(20) = 1460 bytes
  - VPN/터널: MTU 감소로 MSS도 작아짐 (tcpdump에서 mss 값 확인)

Window Scale (RFC 1323):
  - rwnd는 16bit → 최대 65535 bytes
  - Window Scale 옵션: rwnd << scale_factor
  - 최대 scale=14 → rwnd 최대 1GB
  - SYN 단계에서만 협상됨

SACK (Selective Acknowledgment):
  - SYN 단계에서 SACK Permitted 협상
  - 수신자가 받은 블록을 명시 → 송신자가 빈 구간만 재전송
  - 네트워크 손실 많을 때 효율 대폭 향상

TCP Timestamps (RFC 1323):
  - RTT 정밀 측정 (RTTM)
  - PAWS (Protection Against Wrapped Sequences) 제공
  - 보안 상 IP spoofing 탐지에 활용됨
```

```bash
# 연결의 TCP 옵션 실시간 확인
ss -tin dst <remote_ip>
# 출력 예:
# cubic wscale:7,7 rto:204 rtt:4.321/0.5 ato:40 mss:1448
# rcvmss:1448 advmss:1448 cwnd:10 ssthresh:7 bytes_sent:...
# sack ts ecn ecnseen

# tcpdump로 SYN 패킷의 TCP Options 확인
tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-syn != 0' -v | grep -A5 "Flags \[S\]"
```

---

## 3. 3-Way Handshake 커널 내부 동작

### 전체 흐름

```
Client (connect())                        Server (listen())
      │                                        │
      │ 1. ISN_c 생성 (랜덤)                   │
      │    SYN Queue 없음 (클라이언트)          │
      │                                        │
      │──── SYN(seq=ISN_c) ────────────────────▶│
      │                                        │ 2. SYN 수신
      │                                        │    ISN_s 생성
      │                                        │    half-open 연결 → SYN Queue 삽입
      │                                        │    상태: SYN_RCVD
      │                                        │
      │◀─── SYN+ACK(seq=ISN_s, ack=ISN_c+1) ──│
      │                                        │
      │ 3. ACK 전송                             │
      │    상태: ESTABLISHED                    │
      │──── ACK(seq=ISN_c+1, ack=ISN_s+1) ─────▶│
                                               │ 4. ACK 수신
                                               │    SYN Queue → Accept Queue 이동
                                               │    상태: ESTABLISHED
                                               │    app이 accept() 호출 대기
```

### ISN (Initial Sequence Number) 생성

```
ISN은 랜덤하게 생성 (보안상 예측 불가해야 함)

커널 구현 (Linux):
- secure_tcp_seq() 함수
- SHA-1 기반 해시: f(srcIP, dstIP, srcPort, dstPort, secret_key)
- + 타이머 기반 카운터 (4µs마다 증가)

왜 랜덤인가?
- ISN 예측 공격 방어 (Session Hijacking)
- TCP Sequence 예측으로 RST 주입 방지
```

### SYN Queue와 Accept Queue

```
                         listen() 호출
                              │
              ┌───────────────┴───────────────┐
              │                               │
         SYN Queue                       Accept Queue
    (net.ipv4.tcp_max_syn_backlog)    (net.core.somaxconn)
              │                               │
         SYN_RCVD 상태               ESTABLISHED 상태
         (미완료 연결)                 (완료 연결, app 대기)
              │                               │
     3-way handshake 완료 ──────────────────▶ │
                                              │
                                        app accept() ◀── 블로킹 대기
```

```bash
# SYN Queue 오버플로 확인
netstat -s | grep "SYNs to LISTEN"
# "123 SYNs to LISTEN sockets dropped" → SYN Queue 부족

# Accept Queue 오버플로 확인
netstat -s | grep "times the listen queue"
# "5 times the listen queue of a socket overflowed"

# 현재 큐 크기 파라미터
sysctl net.ipv4.tcp_max_syn_backlog   # SYN Queue (기본 1024)
sysctl net.core.somaxconn             # Accept Queue 상한 (기본 4096)

# 실제 큐 사용량 확인 (Recv-Q = Accept Queue 대기 연결 수)
ss -lnt
# State    Recv-Q Send-Q   Local Address:Port
# LISTEN   0      128      0.0.0.0:80         ← Recv-Q가 커지면 앱이 accept() 못 따라감
```

### SYN Cookies 동작 원리

```
SYN Flood: 대량의 SYN만 보내고 ACK 안 함 → SYN Queue 고갈

SYN Cookie 해결책:
  1. SYN 수신 시 SYN Queue에 저장하지 않음
  2. 대신 SYN+ACK의 ISN_s를 특수 쿠키로 계산
     cookie = f(srcIP, dstIP, srcPort, dstPort, timestamp, secret)
  3. 클라이언트의 ACK(ack=cookie+1) 수신 시 쿠키 검증
  4. 검증 성공 시 Accept Queue에 바로 삽입

단점:
  - TCP Options(SACK, Window Scale, Timestamps) 협상 불가
  - → 연결 성능 저하 가능성
  - syncookies 활성화 상태에서도 SYN Queue 미초과 시 일반 방식 사용

sysctl net.ipv4.tcp_syncookies=1   # 1=SYN 큐 가득 찰 때만 활성화 (권장)
```

---

## 4. 4-Way Handshake 심화

### 정상 종료 흐름

```
Active Closer (먼저 close())              Passive Closer
        │                                       │
        │ close() 호출 → FIN 전송               │
        │──── FIN(seq=M) ───────────────────────▶│
        │                                       │ FIN 수신 → CLOSE_WAIT
        │                                       │ 앱에 EOF 전달 (read() 반환 0)
        │◀─── ACK(ack=M+1) ─────────────────────│
        │ FIN_WAIT_2 상태                        │ 앱이 남은 데이터 전송 가능 (Half-Close)
        │                                       │
        │                                       │ close() 호출 → FIN 전송
        │◀─── FIN(seq=N) ────────────────────── │
        │ TIME_WAIT 상태                         │ LAST_ACK 상태
        │                                       │
        │──── ACK(ack=N+1) ─────────────────────▶│
        │                                       │ CLOSED
        │ 2MSL 대기 후 CLOSED                    │
```

### TIME_WAIT 존재 이유 (면접 핵심)

```
이유 1: 마지막 ACK 유실 대비
  Active Closer가 보낸 마지막 ACK가 유실되면
  Passive Closer가 FIN을 재전송한다.
  TIME_WAIT 없이 CLOSED 되면 새 연결에서 엉뚱한 FIN을 받게 됨.
  → TIME_WAIT = MSL(Maximum Segment Lifetime) × 2 = 보통 60초 (Linux 기본)

이유 2: Delayed Segment 방지
  이전 연결의 지연된 패킷이 같은 4-tuple(srcIP:port, dstIP:port)의
  새 연결로 흘러들어가는 것 방지.
  MSL 동안 네트워크상의 모든 패킷이 소멸됨을 보장.

Linux에서 MSL 설정:
  /proc/sys/net/ipv4/tcp_fin_timeout = 60
  (TIME_WAIT 지속 시간. 실제 2MSL이 아닌 커스텀 가능)
```

### CLOSE_WAIT 고착 (실무 장애 패턴)

```
문제:
  서버에 CLOSE_WAIT이 수백~수천 개 → 파일 디스크립터 고갈 → 새 연결 불가

원인:
  클라이언트가 FIN을 보냄 → 서버 커널이 ACK 응답 → CLOSE_WAIT
  이후 서버 앱이 close()를 호출하지 않음
  → 앱 코드 버그: try-catch에서 소켓 close 누락, 커넥션 풀 미반환 등

진단:
  ss -tan state close-wait | awk '{print $4}' | cut -d: -f1 | sort | uniq -c

해결:
  앱 코드 수정 (finally 블록에서 close() 보장)
  Java: try-with-resources 사용
  Go: defer conn.Close()
  임시 방편: 해당 프로세스 재시작
```

### RST 패킷 시나리오

```
RST를 보내는 케이스:
  1. 존재하지 않는 포트에 SYN → 커널이 RST 응답
  2. 연결 중 비정상 종료 (프로세스 kill -9)
  3. 방화벽이 패킷 차단 후 RST 주입
  4. Half-Open 연결 감지 (한쪽만 ESTABLISHED)
  5. SO_LINGER(l_onoff=1, l_linger=0)으로 close() → RST 전송

FIN vs RST:
  FIN: "더 이상 보낼 데이터 없음" (정상 종료, Half-Close 가능)
  RST: "이 연결은 비정상, 즉시 폐기" (데이터 손실 가능)

tcpdump로 RST 탐지:
  tcpdump -nn 'tcp[tcpflags] & tcp-rst != 0'
```

---

## 5. 흐름 제어 (Flow Control) 심화

### 슬라이딩 윈도우 동작 원리

```
송신자 버퍼:
┌──────────────────────────────────────────────────────────────┐
│ 전송&ACK됨 │ 전송&ACK 미수신 │ 전송 가능   │ 전송 불가    │
│  (폐기됨)  │  (재전송 대기)  │ (윈도우 내) │ (rwnd 초과)  │
└──────────────────────────────────────────────────────────────┘
             ↑                              ↑
          SND.UNA                       SND.UNA + min(rwnd, cwnd)
```

```
수신자 → 송신자에게 rwnd 광고:
  TCP 헤더의 Window Size 필드 = 수신 버퍼 여유 공간

송신자 전송 가능량:
  effective_window = min(rwnd, cwnd) - bytes_in_flight

윈도우가 이동(slide)하는 조건:
  ACK 수신 시 → SND.UNA 전진 → 왼쪽 경계 이동
  rwnd 증가 시 → 오른쪽 경계 이동
```

### Window Scaling (RFC 1323)

```
문제: TCP Window Size는 16bit → 최대 65,535 bytes
     BDP = 10Gbps × 100ms RTT = 125MB → window가 너무 작아 대역폭 낭비

해결: TCP Options의 Window Scale Factor
  실제 rwnd = Window Size × 2^(scale_factor)
  scale_factor 최대 14 → 최대 window = 65535 × 2^14 ≈ 1GB

협상: SYN/SYN+ACK 단계에서만 가능
  한쪽이라도 Window Scale을 포함하지 않으면 적용 안 됨

확인:
  ss -tin | grep wscale
  # wscale:7,7 → 양방향 모두 2^7=128배 스케일
  # 실제 rwnd = 수신된 window_size × 128
```

### Zero Window와 Window Probe

```
Zero Window (흐름 제어의 핵심 병목):
  수신 버퍼가 가득 찬 경우 rwnd=0 광고
  → 송신자는 전송 중단

Window Probe:
  rwnd=0 수신 후 송신자는 주기적으로 1 byte 데이터 전송
  → 수신자의 업데이트된 rwnd를 얻기 위함
  → tcp_probe_interval로 주기 조정

Wireshark에서 보이는 패턴:
  [TCP ZeroWindow] → [TCP ZeroWindowProbe] → [TCP ZeroWindowProbeAck] 반복
  → 수신자 앱이 데이터를 처리하지 못함 (앱 처리 병목)
  → 해결: 앱 처리 속도 개선, 수신 버퍼 확대
```

### Silly Window Syndrome

```
문제:
  수신자가 1 byte만 빈 윈도우를 광고 → 송신자가 1 byte짜리 패킷 전송 반복
  → TCP 헤더(40byte) 대비 데이터 비율 극히 낮음 → 효율 폭락

해결:
  수신자 측: Clark's Algorithm
    rwnd < min(MSS/2, 수신버퍼/2) 이면 rwnd=0 광고 (충분히 찰 때까지 대기)
  송신자 측: Nagle 알고리즘
    작은 데이터를 ACK 수신 전까지 버퍼링 → 한 번에 전송
```

---

## 6. 혼잡 제어 (Congestion Control) 심화

### CWND vs rwnd

```
rwnd: 수신자가 허용하는 최대 미확인 데이터량 (수신 버퍼 기반)
cwnd: 송신자가 네트워크 혼잡에 따라 자체 제한하는 전송량
실제 전송: bytes_in_flight ≤ min(rwnd, cwnd)
```

### 상태 전환 상세

```
[초기 연결]
  cwnd = IW (Initial Window) = 10 MSS (RFC 6928, 현재 Linux 기본)
  ssthresh = 65535 (초기값, 사실상 무한대)

[Slow Start]
  매 ACK마다: cwnd += 1 MSS
  → RTT당 cwnd ≈ 2배 (지수 증가)
  → cwnd ≥ ssthresh가 되면 Congestion Avoidance 전환

[Congestion Avoidance]
  매 ACK마다: cwnd += MSS × MSS / cwnd
  → RTT당 cwnd += 1 MSS (선형 증가)

[혼잡 감지: 3 Duplicate ACK]
  TCP Reno:
    ssthresh = cwnd / 2
    cwnd = ssthresh (Fast Recovery 진입)
    → Congestion Avoidance 재개 (Slow Start 스킵)

  TCP Tahoe (구식):
    ssthresh = cwnd / 2
    cwnd = 1 MSS (Slow Start 재시작)

[혼잡 감지: Timeout (RTO 만료)]
  ssthresh = cwnd / 2
  cwnd = 1 MSS
  → Slow Start 재시작 (가장 공격적인 감소)
```

### Fast Retransmit & Fast Recovery

```
3 Duplicate ACK 발생 시:
  1. 타임아웃 기다리지 않고 즉시 손실 세그먼트 재전송 (Fast Retransmit)
  2. cwnd = ssthresh = cwnd/2 (Fast Recovery)
  3. 재전송 후 수신되는 각 ACK마다 cwnd += 1 MSS (임시 증가)
  4. 새로운 데이터에 대한 ACK 수신 시 Fast Recovery 종료
  5. cwnd = ssthresh → Congestion Avoidance 재개

SACK 활성화 시:
  손실된 세그먼트만 정확히 재전송 (중복 전송 없음)
  → 고손실 환경에서 성능 대폭 향상
```

### CUBIC 알고리즘 (Linux 기본)

```
목표: 고대역폭 고지연 네트워크(LFN)에서 CWND 빠르게 회복

Window 증가 함수 (cubic 곡선):
  W(t) = C × (t - K)³ + W_max

  K: cwnd가 W_max에 도달하는 시간
  W_max: 혼잡 직전 cwnd
  C: 스케일링 상수 (기본 0.4)

특성:
  - W_max 근처: 천천히 증가 (조심스럽게 탐색)
  - W_max 이후: 빠르게 증가 (대역폭 적극 활용)
  - RTT 독립적 (Reno는 RTT가 길면 불리함)
  - ECN 지원
```

### BBR (Bottleneck Bandwidth and RTT)

```
패러다임 전환: 손실 기반 → 모델 기반

핵심 아이디어:
  네트워크는 두 가지 자원이 병목:
  1. BtlBw (Bottleneck Bandwidth): 최대 처리 가능 대역폭
  2. RTprop (Round-trip propagation time): 최소 RTT (큐 없는 상태)

목표:
  패킷이 BtlBw 속도로 전송되면서 큐는 최소화
  (손실 없이 최대 대역폭 활용)

BBR 상태 기계:
  STARTUP: 대역폭 측정. 2 × 증가율로 급격히 증가
  DRAIN: STARTUP 과잉 전송 드레인. 큐 비움
  PROBE_BW: 정상 운전 모드. 주기적으로 대역폭 탐색
  PROBE_RTT: 주기적으로 cwnd 최소화 → 정확한 RTprop 측정

CUBIC vs BBR 비교:
  패킷 손실 5% 환경: BBR >> CUBIC (손실을 혼잡으로 보지 않음)
  LAN 환경: 비슷하거나 CUBIC이 유리한 경우도 있음
  위성/WAN: BBR이 명확히 우수
  공정성: BBR은 CUBIC과 공존 시 불공정할 수 있음 (BBR끼리는 공정)
```

### ECN (Explicit Congestion Notification)

```
기존: 라우터가 버퍼 가득 → 패킷 드롭 → 손실로 혼잡 감지 (손실 발생 후 감지)
ECN: 라우터가 버퍼 가득 → 패킷에 CE(Congestion Experienced) 마크 → 수신자가 ECE 플래그로 알림

흐름:
  1. SYN/SYN+ACK에서 ECN 지원 협상 (ECE+CWR 플래그)
  2. 라우터가 혼잡 → IP 헤더 ECN 필드에 CE 마크
  3. 수신자 → ACK에 ECE 플래그 set
  4. 송신자 → CWR 플래그 set + cwnd 감소 (손실 없이!)

설정:
  sysctl net.ipv4.tcp_ecn=1   # 1=ECN 요청, 2=항상 ECN, 0=비활성화
  # 일부 구형 장비에서 ECN 패킷 드롭 이슈 → 환경 확인 후 활성화
```

---

## 7. TCP 재전송 메커니즘

### RTO (Retransmission Timeout) 계산

```
SRTT (Smoothed RTT):
  SRTT = (1 - α) × SRTT + α × RTT_sample    (α = 1/8)

RTTVAR (RTT Variance):
  RTTVAR = (1 - β) × RTTVAR + β × |SRTT - RTT_sample|   (β = 1/4)

RTO:
  RTO = SRTT + 4 × RTTVAR
  최솟값: 200ms (Linux), 최댓값: 120초

RTO Backoff (지수 증가):
  재전송 시 RTO = RTO × 2
  tcp_retries2 횟수 초과 시 연결 종료 (기본 15회 → 약 15분)
```

```bash
# 현재 RTO 확인
ss -tin | grep rto
# rto:204 rtt:4.321/0.5 → RTO=204ms, RTT=4.321ms

# 재전송 횟수 설정
sysctl net.ipv4.tcp_retries1   # 기본 3: 이후 IP 레이어에 문제 알림
sysctl net.ipv4.tcp_retries2   # 기본 15: 이후 연결 포기
sysctl net.ipv4.tcp_syn_retries  # 기본 6: SYN 재전송 횟수

# 재전송 통계
netstat -s | grep -i retransmit
ss -s
```

### Karn's Algorithm

```
문제: 재전송한 패킷의 ACK가 원본의 ACK인지 재전송의 ACK인지 모름
     → 잘못된 RTT 샘플로 RTO 왜곡

해결:
  재전송된 세그먼트의 ACK는 RTT 계산에 사용하지 않음
  → Timestamps 옵션(RTTM)으로 해결 가능
     (패킷마다 타임스탬프 → 재전송 여부와 무관하게 정확한 RTT 측정)
```

---

## 8. VPC Flow Logs와 TCP 분석

### VPC Flow Log 필드 구조

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

예:
2 123456789010 eni-abc12345 10.0.1.100 10.0.2.200 54321 443 6 10 5120 1620000000 1620000060 ACCEPT OK
                                                                         ↑
                                                        protocol=6 → TCP (17=UDP, 1=ICMP)
```

### Flow Log의 한계와 이해

```
VPC Flow Log는 TCP 연결 전체를 기록하지 않음
→ 집계(aggregation) 방식: 기본 10초 또는 60초 단위로 묶어서 기록

한 연결이 여러 레코드로 나뉠 수 있음:
  start=0s  ~ 60s: ACCEPT, packets=100
  start=60s ~ 120s: ACCEPT, packets=200

Flow Log에는 TCP 플래그(SYN, FIN, RST)가 없음!
→ 연결 성립 여부는 action(ACCEPT/REJECT)으로만 판단
→ 실제 TCP 상태는 알 수 없음 (Wireshark나 tcpdump 필요)
```

### 장애 시나리오별 Flow Log 패턴

**시나리오 1: 보안 그룹이 차단하는 경우**
```
클라이언트 → 서버 SYN: REJECT 기록
서버 → 클라이언트: 기록 없음 (패킷 자체가 도달 안 함)

증상:
  클라이언트: 연결 타임아웃 (SYN 재전송 후 포기)
  서버: Flow Log 없음 (수신 자체 안 됨)

Flow Log:
  srcaddr=client, dstaddr=server, dstport=443, action=REJECT

확인:
  aws ec2 describe-security-groups → 인바운드 규칙 확인
  aws ec2 describe-network-acls → NACL 확인 (NACL은 stateless!)
```

**시나리오 2: 서버가 RST 응답**
```
SYN: ACCEPT (도달했음)
서버가 RST: ACCEPT (RST도 Flow Log에는 ACCEPT로 찍힘)

Flow Log:
  srcaddr=server, dstaddr=client, action=ACCEPT, packets=1, bytes=40 (RST 패킷)

증상: 클라이언트 즉시 "Connection refused"
원인: 포트가 열려 있지 않음 (프로세스 미실행), iptables REJECT 규칙

확인:
  서버에서 ss -lnt | grep <port>
  서버에서 iptables -L -n | grep REJECT
```

**시나리오 3: 비대칭 패킷 (Flow Log 레코드 불완전)**
```
NACL은 stateless → 인바운드/아웃바운드 각각 규칙 필요
아웃바운드 NACL이 막으면:
  인바운드: ACCEPT 기록
  아웃바운드: REJECT 기록 (응답 패킷 차단)

증상: SYN은 도달하지만 SYN+ACK가 클라이언트에 도달 안 함
     클라이언트: 연결 타임아웃
     서버: 연결 성립된 것처럼 보임 (ACK 미수신 → 재전송)
```

**시나리오 4: Connection Tracking과 Flow Log**
```
ESTABLISHED 연결 중 보안 그룹 변경:
  보안 그룹은 stateful → 기존 ESTABLISHED 연결은 유지됨
  NACL은 stateless → 기존 연결도 새 규칙 즉시 적용

Flow Log에서 기존 연결이 REJECT로 전환되면 → NACL 변경 의심
```

### CloudWatch Logs Insights 쿼리 예시

```sql
-- 특정 IP와의 REJECT 이벤트 집계
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, action
| filter action = "REJECT" and dstAddr = "10.0.2.200"
| stats count() as rejects by srcAddr, dstPort
| sort rejects desc

-- 포트별 트래픽 볼륨 분석
fields dstPort, bytes, packets
| filter action = "ACCEPT"
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets by dstPort
| sort totalBytes desc

-- 연결 시도 실패율 (REJECT / 전체)
fields action
| stats count() as total by action
```

### AWS VPC Flow Logs v5 (신규 필드)

```
v5에서 추가된 필드:
  pkt-srcaddr, pkt-dstaddr: NAT 이후의 실제 IP (NAT 환경에서 원본 IP 추적)
  flow-direction: ingress / egress
  traffic-path: 1=IGW, 2=NAT GW, 3=Transit GW, 4=VGW, ...

활용:
  NAT Gateway 뒤의 실제 출발지 IP 추적 가능
  트래픽 경로 분석으로 비용 최적화 (데이터 전송 비용)
```

---

## 9. 커널 파라미터 완전 정리 (sysctl)

### 연결 수립 관련

```bash
# SYN 재전송 횟수 (기본 6 → 약 127초 대기)
sysctl net.ipv4.tcp_syn_retries=3      # 빠른 실패 필요 시 줄임

# SYNACK 재전송 횟수 (서버 측, 기본 5)
sysctl net.ipv4.tcp_synack_retries=2

# SYN Queue 크기
sysctl net.ipv4.tcp_max_syn_backlog=65536

# Accept Queue 상한
sysctl net.core.somaxconn=65536

# SYN Cookies
sysctl net.ipv4.tcp_syncookies=1      # 1: SYN 큐 가득 찰 때만
```

### 버퍼 & 윈도우 관련

```bash
# BDP 계산: Bandwidth × RTT
# 예: 10Gbps × 50ms = 62.5MB → 버퍼를 이 이상으로 설정

# 자동 튜닝 (기본 활성화)
sysctl net.ipv4.tcp_moderate_rcvbuf=1

# 소켓 수신 버퍼: min / default / max
sysctl -w net.ipv4.tcp_rmem="4096 131072 67108864"   # max=64MB
sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"

# 커널 소켓 버퍼 절대 최대 (tcp_rmem max보다 크거나 같아야 함)
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.wmem_max=67108864

# Window Scaling 활성화 (기본 on)
sysctl net.ipv4.tcp_window_scaling=1

# 기본 소켓 수신/송신 버퍼 (비 TCP 소켓)
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144
```

### TIME_WAIT 관련

```bash
# TIME_WAIT 소켓 최대 수 (초과 시 오래된 소켓 강제 파괴)
sysctl net.ipv4.tcp_max_tw_buckets=1048576

# TIME_WAIT 소켓 재사용 (클라이언트 측 아웃바운드 연결)
# 같은 4-tuple에 대해 TIME_WAIT 소켓 포트 재사용
sysctl net.ipv4.tcp_tw_reuse=1        # Timestamps 옵션과 함께 사용 (안전)
# tcp_tw_recycle은 Linux 4.12에서 제거됨 (NAT 환경 패킷 드롭 버그)

# FIN_WAIT_2 타임아웃 (기본 60초)
sysctl net.ipv4.tcp_fin_timeout=30    # 줄여서 빠른 정리

# 로컬 포트 범위 (클라이언트 아웃바운드)
sysctl -w net.ipv4.ip_local_port_range="10000 65535"  # 55535개
```

### Keep-Alive 관련

```bash
# SO_KEEPALIVE 옵션 설정 소켓에 적용됨

# 유휴 연결에서 Keepalive 프로브 시작까지 대기 시간 (기본 7200초=2시간)
sysctl -w net.ipv4.tcp_keepalive_time=60

# 프로브 간격 (기본 75초)
sysctl -w net.ipv4.tcp_keepalive_intvl=10

# 프로브 횟수 (기본 9회)
sysctl -w net.ipv4.tcp_keepalive_probes=6
# 위 설정: 60초 유휴 → 10초 간격 6번 → 응답 없으면 연결 종료 (총 120초)
```

### 혼잡 제어 관련

```bash
# 혼잡 제어 알고리즘 확인 및 변경
sysctl net.ipv4.tcp_congestion_control          # 현재
sysctl net.ipv4.tcp_available_congestion_control # 사용 가능 목록

# BBR 활성화 (커널 4.9+)
modprobe tcp_bbr
sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl -w net.core.default_qdisc=fq             # BBR과 함께 FQ 필수

# CUBIC 유지 (기본)
sysctl -w net.ipv4.tcp_congestion_control=cubic

# ECN 활성화
sysctl -w net.ipv4.tcp_ecn=1

# 초기 CWND/RWND 설정 (라우팅 수준)
ip route change default via <gw> initcwnd 10 initrwnd 10

# SACK 활성화 (기본 on)
sysctl net.ipv4.tcp_sack=1
# DSACK (중복 수신 알림)
sysctl net.ipv4.tcp_dsack=1
```

### 재전송 관련

```bash
# 연결 포기까지 재전송 횟수 (기본 15, 약 15분)
sysctl net.ipv4.tcp_retries2=8   # 줄이면 빠르게 포기 (장애 전파 방지)

# 초기 SYN 재전송 횟수 (기본 6)
sysctl net.ipv4.tcp_syn_retries=3

# Orphan 소켓 (close() 후 FIN 전송 중인 소켓) 최대 수
sysctl net.ipv4.tcp_max_orphans=65536
```

### 보안 관련

```bash
# RFC 1337 TIME_WAIT 보호 (RST로 TIME_WAIT 종료 방지)
sysctl net.ipv4.tcp_rfc1337=1

# TCP Timestamps (PAWS, RTTM에 활용. 끄면 Sequence Wrap 취약)
sysctl net.ipv4.tcp_timestamps=1   # 1=활성화 (권장)
# 주의: 일부에서 정보 노출 우려로 끄기도 하지만 현대 커널에서는 랜덤화됨

# SYN Cookies (SYN Flood 방어)
sysctl net.ipv4.tcp_syncookies=1

# 소스 라우팅 비활성화
sysctl net.ipv4.conf.all.accept_source_route=0

# ICMP redirect 비활성화 (라우팅 변조 방어)
sysctl net.ipv4.conf.all.accept_redirects=0
sysctl net.ipv4.conf.all.send_redirects=0
```

### conntrack 관련

```bash
# 연결 추적 테이블 최대 엔트리 수
sysctl net.netfilter.nf_conntrack_max=1048576          # 기본 65536
sysctl net.netfilter.nf_conntrack_buckets=262144       # max의 1/4

# 현재 사용량 확인
cat /proc/sys/net/netfilter/nf_conntrack_count
# 이 값이 max에 근접하면 → 새 연결 드롭 → 심각한 장애

# 상태별 타임아웃 튜닝
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600   # 기본 432000(5일!)
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=120
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=60
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_fin_wait=120
```

### sysctl 설정 영구 적용

```bash
# /etc/sysctl.d/99-tcp-tuning.conf
cat > /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
# Connection setup
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_max_syn_backlog = 65536
net.core.somaxconn = 65536
net.ipv4.tcp_syncookies = 1

# Buffers (for 10Gbps)
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 131072 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_moderate_rcvbuf = 1

# TIME_WAIT
net.ipv4.tcp_max_tw_buckets = 1048576
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.ip_local_port_range = 10000 65535

# Keep-Alive
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6

# Congestion control
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq
net.ipv4.tcp_ecn = 1
net.ipv4.tcp_sack = 1

# Retransmission
net.ipv4.tcp_retries2 = 8

# Security
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_timestamps = 1

# conntrack
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 600
EOF

# 적용
sysctl -p /etc/sysctl.d/99-tcp-tuning.conf

# 확인
sysctl -a | grep tcp | grep -v "#"
```

---

## 10. 실무 진단 도구 모음

### ss — 소켓 통계 (netstat 대체)

```bash
# 모든 TCP 소켓 (Listening 포함)
ss -tan

# 상세 정보 (혼잡 제어, 타이머 등)
ss -tin

# 상태별 필터
ss -tan state established
ss -tan state time-wait
ss -tan state close-wait

# 특정 포트
ss -tan sport = :443
ss -tan dport = :443

# 프로세스 정보 포함
ss -tanp

# 소켓 통계 요약
ss -s

# 타이머 정보 (재전송 타이머, Keepalive 타이머)
ss -to
```

### tcpdump — 패킷 캡처

```bash
# 특정 호스트와의 TCP 트래픽
tcpdump -nn -i eth0 host 10.0.1.100 and tcp

# SYN 패킷만 (연결 시도 모니터링)
tcpdump -nn 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'

# RST 패킷 탐지
tcpdump -nn 'tcp[tcpflags] & tcp-rst != 0'

# 포트 443, 헤더와 첫 100바이트 캡처
tcpdump -nn -i eth0 tcp port 443 -s 100

# pcap 파일로 저장 (Wireshark 분석용)
tcpdump -nn -i eth0 -w /tmp/capture.pcap

# Zero Window 탐지 (window size = 0)
tcpdump -nn 'tcp[14:2] == 0 and tcp[tcpflags] & tcp-ack != 0'
```

### /proc/net/tcp 파싱

```bash
# 형식: sl  local_address rem_address st tx_queue rx_queue ...
# (16진수, 리틀 엔디안)
cat /proc/net/tcp | awk '
NR>1 {
  split($2, local, ":")
  split($3, remote, ":")
  printf "Local: %d.%d.%d.%d:%d Remote: %d.%d.%d.%d:%d State: %s\n",
    strtonum("0x"substr(local[1],7,2)),
    strtonum("0x"substr(local[1],5,2)),
    strtonum("0x"substr(local[1],3,2)),
    strtonum("0x"substr(local[1],1,2)),
    strtonum("0x"local[2]),
    strtonum("0x"substr(remote[1],7,2)),
    ...
}'

# 상태 코드:
# 01=ESTABLISHED, 02=SYN_SENT, 03=SYN_RCVD, 04=FIN_WAIT1
# 05=FIN_WAIT2, 06=TIME_WAIT, 07=CLOSE, 08=CLOSE_WAIT
# 09=LAST_ACK, 0A=LISTEN, 0B=CLOSING
```

### 연결 수 모니터링

```bash
# ESTABLISHED 연결 수 (1초 간격)
watch -n1 'ss -s | grep Estab'

# 상태별 카운트
ss -tan | awk 'NR>1{print $1}' | sort | uniq -c | sort -rn

# 목적지 IP별 연결 수
ss -tan state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# netstat 재전송 비율 계산
netstat -s | grep -E "segments send|retransmit"
# retransmit / sent × 100 = 재전송률
# 1% 이하 정상, 5% 이상 → 네트워크 문제 조사
```

### 패킷 손실 및 RTT 측정

```bash
# mtr: traceroute + ping 조합 (경로별 손실률, RTT)
mtr --report --report-cycles 100 <target_ip>

# 특정 연결의 RTT
ss -tin dst <server_ip> | grep rtt

# iperf3: 실제 처리량 측정
# 서버
iperf3 -s -p 5201
# 클라이언트
iperf3 -c <server> -p 5201 -t 30 -P 4   # 4개 스트림, 30초
```

---

## 11. 실무 심화 포인트 & 면접 대비

### "TIME_WAIT이 대량 발생하면 문제인가?"
```
대부분 정상이다. TIME_WAIT은 능동 종료 측(보통 클라이언트)에 발생.
단기 HTTP 요청이 많은 서비스에서는 당연한 현상.

진짜 문제가 되는 경우:
  1. 포트 고갈: TIME_WAIT 소켓이 ephemeral port를 점유
     → tcp_tw_reuse=1 + 포트 범위 확장
  2. 메모리 고갈: tcp_max_tw_buckets 초과 → 오래된 소켓 강제 파괴
     → 수가 너무 많으면 Keep-Alive, 커넥션 풀 도입으로 근본 해결

CLOSE_WAIT이 많은 것은 진짜 문제 (앱 버그).
```

### "CWND와 rwnd의 차이는?"
```
rwnd: 수신자의 버퍼 여유 공간. 수신자가 제어.
cwnd: 네트워크 혼잡 수준에 따른 송신자의 자체 제한. 송신자가 제어.
실제 전송 가능량 = min(rwnd, cwnd) - 미확인 전송량(in-flight)

"Window Full" 메시지 = rwnd 한계
"cwnd limited" 메시지 = cwnd 한계
```

### "SYN 패킷이 도달하는지 확인하는 방법?"
```
서버 측:
  tcpdump -nn 'tcp[tcpflags] & tcp-syn != 0' -i eth0
  → SYN이 캡처되면 도달함. SYN+ACK 전송 여부도 확인.

클라이언트 측:
  tcpdump -nn 'host <server_ip>' -i eth0
  → SYN 전송 확인, SYN+ACK 수신 여부 확인

SYN 보냈는데 SYN+ACK 없음:
  → 서버에 도달하지 않음 (방화벽/라우팅) 또는
  → 서버가 SYN+ACK를 보냈는데 클라이언트에 도달 안 함 (비대칭 라우팅)
```

### "BBR을 항상 써야 하는가?"
```
BBR이 유리한 환경:
  - 고손실 WAN/위성 링크
  - 고지연 (RTT > 50ms) 대역폭 환경
  - CDN 엣지 (다양한 클라이언트 연결)

CUBIC이 유리하거나 동등한 환경:
  - 데이터센터 내부 (저손실, 저지연)
  - 공유 네트워크에서 공정성 중요 시
  - 버퍼가 충분한 LAN

주의: BBR은 버퍼가 작은 환경에서 패킷 버스트를 일으킬 수 있음
     → BBR + FQ(Fair Queue) 조합이 필수
```

### "CLOSE_WAIT 장애 즉시 대응 절차"
```
1. 확인:
   ss -tan state close-wait | wc -l   → 수백 이상이면 문제
   ss -tanp state close-wait | awk '{print $6}' | sort | uniq -c
   → 어느 프로세스/포트인지 파악

2. 원인 분석:
   해당 프로세스의 코드에서 소켓 close() 누락 확인
   Java: Thread Dump로 어느 스레드가 소켓을 잡고 있는지 확인
   Go: pprof goroutine dump

3. 임시 조치:
   해당 프로세스 재시작 (소켓 강제 해제)
   lsof -p <pid> | grep TCP 로 열린 소켓 확인

4. 영구 해결:
   코드 수정: finally / defer / try-with-resources로 close() 보장
   Keep-Alive timeout 설정으로 유휴 연결 정리
```

### "VPC Flow Log에서 연결이 ACCEPT인데 통신이 안 되는 경우"
```
가능한 원인:
  1. NACL은 ACCEPT지만 보안그룹이 다른 방향 차단
     (NACL은 stateless → 인바운드 ACCEPT + 아웃바운드도 별도 규칙 필요)
  2. OS 레벨 iptables/firewalld 차단
  3. 앱이 Listen하지 않음 (RST 응답 → Flow Log에 ACCEPT로 기록됨)
  4. MTU 불일치 (대용량 패킷만 드롭)

MTU 불일치 디버깅:
  ping -M do -s 1472 <target>   # DF bit set, 1500 MTU - 28 = 1472
  → ICMP Need Fragmentation 수신 여부 확인
  → VPN/터널 환경에서 흔함 (MSS Clamping 필요)
  iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
           -j TCPMSS --clamp-mss-to-pmtu
```

### 고빈도 면접 질문 요약

| 질문 | 핵심 답변 |
|------|-----------|
| 3-way handshake 왜 3번인가? | 양방향 ISN 동기화. 2번은 불충분 (서버→클라 ISN 확인 불가) |
| 4-way handshake 왜 4번인가? | Half-Close: 각 방향 독립 종료. FIN은 한 방향만 닫음 |
| TIME_WAIT 목적 | 마지막 ACK 유실 대비 + 지연 패킷 격리 (2MSL) |
| CLOSE_WAIT 원인 | 앱이 close() 미호출. 코드 버그 |
| Slow Start 왜 느린 시작? | 이름이 아이러니. 처음엔 작게 → 빠르게 증가하는 방식 (slow=작게 시작) |
| BBR vs CUBIC | BBR=모델 기반(RTT+BW), CUBIC=손실 기반. 고손실 환경에서 BBR 유리 |
| tcp_tw_reuse vs recycle | reuse=안전(Timestamps 기반 검증), recycle=위험(제거됨, NAT 버그) |
| conntrack 가득 차면? | 새 연결 드롭. nf_conntrack_max 증가 + timeout 축소 |
