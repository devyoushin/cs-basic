# VPN (Virtual Private Network)

## VPN이란

공개 네트워크(인터넷) 위에 암호화된 터널을 만들어 사설 네트워크처럼 통신하는 기술.

```
[클라이언트] ──암호화된 터널──> [VPN 서버/게이트웨이] ──> [사설 네트워크 or 인터넷]
```

**핵심 기능**
1. **기밀성**: 트래픽 암호화 (도청 방지)
2. **무결성**: 데이터 변조 방지
3. **인증**: 연결 주체 신원 확인
4. **터널링**: 원래 패킷을 캡슐화해서 전송

---

## 터널링 모드

### Transport Mode
- IP 헤더는 유지, **페이로드(데이터)만 암호화**
- 엔드-투-엔드 암호화 (호스트 간 직접 통신)
- IPSec AH/ESP에서 사용

```
원본: [IP 헤더 | TCP | 데이터]
결과: [IP 헤더 | ESP 헤더 | TCP | 데이터 (암호화) | ESP 트레일러]
```

### Tunnel Mode
- **IP 헤더 포함 전체 패킷을 암호화** → 새 IP 헤더로 감쌈
- 게이트웨이 간 통신 (Site-to-Site VPN)
- 원본 IP가 숨겨짐

```
원본: [IP 헤더(10.0.0.1) | TCP | 데이터]
결과: [새 IP 헤더(공인IP) | ESP 헤더 | IP 헤더(10.0.0.1) | TCP | 데이터 (암호화)]
```

---

## IPSec (Internet Protocol Security)

네트워크 계층(L3)에서 동작하는 VPN 프로토콜 모음. RFC 4301~4309.

### IPSec 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| **IKE (Internet Key Exchange)** | 키 교환 및 SA 협상 |
| **ESP (Encapsulating Security Payload)** | 암호화 + 무결성 |
| **AH (Authentication Header)** | 무결성만 (암호화 없음, NAT 미지원으로 거의 미사용) |
| **SA (Security Association)** | 단방향 보안 연결 정보 (알고리즘, 키, SPI) |
| **SPD (Security Policy Database)** | 어떤 트래픽에 IPSec 적용할지 규칙 |
| **SAD (Security Association Database)** | 현재 활성 SA 저장 |

### IKEv2 협상 과정 (2단계)

```
[Phase 1 — IKE SA 수립]
  → DH(Diffie-Hellman) 키 교환으로 공유 비밀키 생성
  → 신원 인증 (인증서 or PSK)
  → IKE SA (제어 채널) 완성

[Phase 2 — Child SA (IPSec SA) 수립]
  → 실제 데이터 암호화에 사용할 SA 협상
  → ESP 알고리즘, 키, SPI 결정
  → SA는 단방향 → 양방향 통신에 SA 2개 필요
```

### IKEv2 패킷 교환

```
Initiator                    Responder
    │── IKE_SA_INIT (1) ────>│  DH 파라미터, Nonce 교환
    │<── IKE_SA_INIT (2) ────│  DH 파라미터, Nonce 응답
    │── IKE_AUTH (3) ────────>│  인증 + Child SA 요청 (암호화)
    │<── IKE_AUTH (4) ────────│  인증 + Child SA 수립 (암호화)
    │══════ 데이터 전송 ══════│  ESP로 암호화된 트래픽
```

### ESP 패킷 구조

```
[IP 헤더] [ESP 헤더: SPI(4) + Seq(4)] [암호화 데이터] [패딩] [ESP 트레일러] [ESP ICV(무결성 체크)]
```

- **SPI (Security Parameters Index)**: 어떤 SA로 복호화할지 식별
- **ICV**: HMAC-SHA256 등으로 무결성 검증

### IPSec 알고리즘 (현대 권장)

| 항목 | 권장 | 비권장 |
|------|------|--------|
| 키 교환 | DH Group 19/20 (ECDH) | DH Group 1/2/5 |
| 암호화 | AES-256-GCM | 3DES, DES |
| 무결성 | SHA-256/384 | MD5, SHA-1 |
| 인증 | RSA/ECDSA 인증서, EAP | PSK (약한 패스워드) |

---

## WireGuard

2020년 Linux 커널 5.6에 공식 통합된 현대적 VPN 프로토콜.

### 설계 원칙
- 코드 약 4,000줄 (OpenVPN 70,000+ 줄) → 감사 가능, 취약점 면적 최소화
- 고정된 최신 암호화 알고리즘 → 협상 과정 없음 (Cryptographic Agility 의도적으로 배제)
- 연결 상태 없음 (Stateless) → Roaming 지원 (IP 변경에도 터널 유지)

### Noise Protocol Framework
WireGuard는 Noise_IKpsk2 핸드셰이크 패턴 사용:

```
Initiator                        Responder
    │── Handshake Initiation ────>│
    │   (Ephemeral DH 공개키,      │
    │    암호화된 Static 공개키,    │
    │    Timestamp)                │
    │                             │
    │<── Handshake Response ───────│
    │   (Ephemeral DH 공개키,      │
    │    빈 페이로드 암호화)        │
    │                             │
    │══════ 데이터 전송 ══════════│  ChaCha20-Poly1305 암호화
```

### WireGuard 암호화 알고리즘 (고정)

| 용도 | 알고리즘 |
|------|---------|
| 키 교환 | Curve25519 (ECDH) |
| 대칭 암호화 | ChaCha20-Poly1305 |
| 해시/HMAC | BLAKE2s |
| 키 파생 | HKDF |

### WireGuard 설정 예시

```ini
# /etc/wireguard/wg0.conf (서버)
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32    # 이 IP만 이 피어에서 허용
PersistentKeepalive = 25    # NAT 유지를 위한 keepalive

# /etc/wireguard/wg0.conf (클라이언트)
[Interface]
Address = 10.0.0.2/24
PrivateKey = <client_private_key>
DNS = 10.0.0.1

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0      # 모든 트래픽을 VPN 경유 (Full Tunnel)
# AllowedIPs = 10.0.0.0/24  # 사내 네트워크만 (Split Tunnel)
```

```bash
wg-quick up wg0     # 인터페이스 활성화
wg show             # 상태 확인 (피어, 최근 핸드셰이크, 전송량)
```

### WireGuard 커널 동작
- 커널 모듈(`wireguard.ko`)로 동작 → 유저스페이스 컨텍스트 스위치 없음 → IPSec/OpenVPN 대비 성능 우수
- `wg0` 인터페이스 = 가상 네트워크 인터페이스 → 라우팅 테이블로 제어

---

## OpenVPN (TLS 기반 VPN)

SSL/TLS를 사용하는 유저스페이스 VPN. 방화벽 통과에 유리 (TCP 443 사용 가능).

### 동작 방식
```
[클라이언트]                    [서버]
    │── TLS 핸드셰이크 (제어 채널) ──>│  인증서 + 키 교환
    │<── 터널 파라미터 교환 ──────────│  IP, 라우트 등
    │══ 데이터 채널 (암호화 트래픽) ══│  AES-256-GCM
```

- **제어 채널**: TLS/SSL (신뢰성 필요 → TCP or UDP+재전송)
- **데이터 채널**: 별도 암호화 (AES, ChaCha20)
- **tun 모드**: L3 터널 (IP 패킷)
- **tap 모드**: L2 터널 (이더넷 프레임, 브리지 구성에 사용)

---

## SSL VPN vs IPSec VPN

| 항목 | SSL/TLS VPN | IPSec VPN |
|------|-------------|-----------|
| 계층 | L7 (애플리케이션) | L3 (네트워크) |
| 방화벽 통과 | 쉬움 (443 포트) | 어려움 (UDP 500/4500, ESP 프로토콜 허용 필요) |
| 클라이언트 | 브라우저 or 경량 앱 | OS 내장 or 전용 클라이언트 |
| 성능 | 상대적으로 낮음 | 높음 (커널 레벨) |
| 활용 | Remote Access VPN | Site-to-Site VPN |

---

## VPN 유형

### Remote Access VPN (원격 접속)
```
[재택 근무자] ──인터넷──> [VPN 서버] ──> [사내 네트워크]
```
- 개별 사용자가 사내 네트워크에 안전하게 접속
- WireGuard, OpenVPN, Cisco AnyConnect, GlobalProtect

### Site-to-Site VPN
```
[본사 네트워크] ──IPSec 터널──> [VPN 게이트웨이] ──> [지사 네트워크]
```
- 두 네트워크를 항상 연결 (상시 터널)
- AWS VPN Gateway, Cisco ASA, pfSense

### MPLS VPN (기업 전용선)
- 통신사 MPLS 망을 이용한 사설 네트워크
- 인터넷을 경유하지 않음 → 높은 안정성/성능, 높은 비용
- AWS Direct Connect, Azure ExpressRoute와 유사한 개념

---

## Split Tunneling

```
Full Tunnel: 모든 트래픽 → VPN → 인터넷
Split Tunnel: 사내 트래픽만 → VPN
              인터넷 트래픽 → 직접 연결
```

| | Full Tunnel | Split Tunnel |
|--|-------------|--------------|
| 보안 | 모든 트래픽 감사 가능 | 인터넷 트래픽 비감사 |
| 성능 | VPN 서버 병목 | 사내 트래픽만 VPN 경유 |
| 설정 | `AllowedIPs = 0.0.0.0/0` | `AllowedIPs = 10.0.0.0/8` |
| 활용 | 보안 민감 환경 | 일반 원격 근무 |

---

## NAT Traversal (NAT-T)

사설 IP를 가진 클라이언트(NAT 뒤)가 IPSec 통신 시 발생하는 문제와 해결.

### 문제
- IPSec ESP 프로토콜 헤더에 포트 정보 없음 → NAT 장비가 변환 불가
- AH는 IP 헤더 포함 무결성 → NAT로 IP 바뀌면 검증 실패

### 해결: UDP 캡슐화 (NAT-T, RFC 3948)
```
원본 ESP 패킷을 UDP 4500 포트로 감쌈
[UDP:4500 | ESP | 데이터]
```
- IKE 협상 시 NAT 감지 → 자동으로 NAT-T 활성화
- WireGuard는 처음부터 UDP 기반 → NAT-T 문제 없음
- `PersistentKeepalive`: NAT 테이블 유지를 위해 주기적 패킷 전송

---

## MTU 문제와 해결

VPN 터널은 헤더를 추가해 패킷이 커짐 → MTU 초과 시 단편화 발생

```
이더넷 MTU: 1500 bytes
WireGuard 오버헤드: ~60 bytes
→ WireGuard 인터페이스 MTU: 1420 (권장)

IPSec ESP Tunnel 오버헤드: ~50~70 bytes
→ IPSec 인터페이스 MTU: 1400~1420
```

### 해결 방법
1. **MTU 낮추기**: VPN 인터페이스 MTU를 1420 이하로 설정
2. **MSS Clamping**: TCP SYN 패킷의 MSS 값을 강제로 낮춤
   ```bash
   iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
     -j TCPMSS --set-mss 1360
   ```
3. **PMTUD (Path MTU Discovery)**: ICMP Type 3 Code 4 메시지로 적절한 MTU 자동 협상
   - 방화벽이 ICMP 차단 시 PMTUD 블랙홀 발생 → MSS Clamping으로 해결

---

## VPN 성능 비교

| 프로토콜 | 처리량 | CPU 부하 | 레이턴시 |
|----------|--------|----------|----------|
| WireGuard | 최고 | 낮음 | 최저 |
| IPSec (커널) | 높음 | 낮음 | 낮음 |
| OpenVPN (UDP) | 중간 | 높음 | 중간 |
| OpenVPN (TCP) | 낮음 | 높음 | 높음 |

**WireGuard 성능 이유**:
- 커널 내 동작 (유저스페이스 전환 없음)
- ChaCha20: AES 하드웨어 가속 없는 환경에서도 빠름
- 간단한 코드베이스 → 최적화 용이

---

## Zero Trust Network Access (ZTNA) — VPN의 대안

전통 VPN: 일단 접속하면 사내 네트워크 전체에 접근 가능 (Flat Network 위험)

ZTNA: 인증된 사용자가 **특정 애플리케이션**에만 접근
```
[사용자] → [Identity Provider (IdP) 인증]
         → [ZTNA 컨트롤러]
         → [특정 앱만 접근 허용] (다른 리소스는 보이지도 않음)
```

| | 전통 VPN | ZTNA |
|--|---------|------|
| 신뢰 모델 | 네트워크 내부 = 신뢰 | 모든 접근 검증 |
| 접근 범위 | 전체 네트워크 | 특정 앱 |
| 사용성 | 클라이언트 필요 | 브라우저 기반 가능 |
| 예시 | OpenVPN, IPSec | Cloudflare Access, Zscaler |

---

## 진단 명령어

```bash
# WireGuard 상태 확인
wg show
wg show wg0 latest-handshakes   # 마지막 핸드셰이크 시간
wg show wg0 transfer            # 전송량 통계

# IPSec 상태 확인 (strongSwan)
ipsec status
ipsec statusall
swanctl --list-sas              # IKEv2 SA 목록

# 터널 인터페이스 확인
ip addr show wg0
ip route show table main | grep wg0

# MTU 확인
ip link show wg0 | grep mtu
ping -M do -s 1400 10.0.0.1    # PMTUD 테스트 (단편화 금지 + 크기 지정)

# 패킷 캡처 (ESP 프로토콜)
tcpdump -i eth0 esp
tcpdump -i eth0 udp port 51820  # WireGuard
tcpdump -i eth0 udp port 500 or udp port 4500  # IKE/NAT-T
```

---

## 실무 포인트

- **WireGuard 우선 선택**: 신규 구축이라면 단순성, 성능, 보안 모두 WireGuard가 우수
- **MTU 설정 필수**: VPN 도입 후 TCP는 되는데 특정 앱이 안 되면 MTU/MSS 문제 의심
- **Key Rotation**: WireGuard는 키 롤오버 자동화 필요 (기본 제공 안 함). `wg set` 명령어나 별도 관리 도구 활용
- **Split Tunnel 기본값**: 재택 근무 VPN은 Split Tunnel로 구성 → VPN 서버 부하 및 병목 방지
- **NAT Keepalive**: 클라이언트가 NAT 뒤에 있으면 `PersistentKeepalive = 25` 설정으로 연결 유지
- **로그 및 감사**: VPN 접속 로그 (IP, 시간, 사용자)는 보안 감사 필수 → SIEM 연동
- **MFA 연동**: OpenVPN + LDAP + TOTP 조합이나 Cloudflare Access + IdP로 다중 인증 강제
- **Zero Trust 전환 고려**: 전통 VPN은 Lateral Movement 공격에 취약 → 규모가 커지면 ZTNA 전환 검토
