# DNS (Domain Name System)

## DNS 조회 흐름

```
브라우저 캐시 → OS 캐시 → /etc/hosts → Stub Resolver (OS)
    → Recursive Resolver (ISP/8.8.8.8)
        → Root NS (13개 클러스터, Anycast)
        → TLD NS (.com, .kr 등)
        → Authoritative NS (도메인 소유자가 운영)
    → 응답 캐싱 (TTL 만큼)
```

### Recursive vs Iterative Resolution

**Recursive (클라이언트 ↔ Recursive Resolver)**
- 클라이언트는 Recursive Resolver에 질의 한 번만 함
- Resolver가 Root → TLD → Auth NS 순서로 대신 쫓아다님
- 응답을 TTL 동안 캐싱 → 이후 쿼리는 캐시에서 바로 응답

**Iterative (Recursive Resolver ↔ 각 NS)**
- Root NS: "모르겠고, `.com` NS는 여기야"
- TLD NS: "모르겠고, `example.com` Auth NS는 여기야"
- Auth NS: "A 레코드는 1.2.3.4야"

```
클라이언트 ──[recursive]──> Recursive Resolver
                                   │ [iterative]
                                   ├──> Root NS → "TLD NS 주소"
                                   ├──> TLD NS  → "Auth NS 주소"
                                   └──> Auth NS → "IP 주소"
```

---

## DNS 메시지 구조 (UDP/TCP 53)

```
+------------------+
| Header (12 bytes) |  QR, Opcode, AA, TC, RD, RA, RCODE
+------------------+
| Question          |  QNAME, QTYPE, QCLASS
+------------------+
| Answer            |  RR 목록 (정답)
+------------------+
| Authority         |  위임 NS 정보
+------------------+
| Additional        |  Glue Record 등 보조 정보
+------------------+
```

- **TC (Truncated)**: UDP 512바이트 초과 시 TCP로 재시도 (EDNS0으로 4096까지 확장 가능)
- **AA (Authoritative Answer)**: Auth NS가 직접 응답했음을 표시
- **RD (Recursion Desired)**: 클라이언트가 재귀 조회 요청
- **RA (Recursion Available)**: 서버가 재귀 조회 지원

### Glue Record
- Auth NS의 도메인이 자기 자신의 Zone에 있을 때 생기는 닭-달걀 문제 해결
- `example.com`의 NS가 `ns1.example.com`이면, `ns1.example.com`의 IP를 Additional 섹션에 포함

---

## DNS 레코드 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| A | 도메인 → IPv4 | `api.example.com → 1.2.3.4` |
| AAAA | 도메인 → IPv6 | `api.example.com → ::1` |
| CNAME | 도메인 → 도메인 (별칭) | `www → example.com` |
| MX | 메일 서버 | `example.com → mail.example.com` (우선순위 포함) |
| TXT | 임의 텍스트 | SPF, DKIM, 도메인 인증 |
| NS | 네임서버 지정 | `example.com → ns1.cloudflare.com` |
| PTR | IP → 도메인 (역방향) | `1.2.3.4 → api.example.com` |
| SOA | Zone의 권한 정보 | 시리얼 번호, 갱신 주기, Negative TTL |
| SRV | 서비스 위치 + 포트 | `_http._tcp.example.com → 80 api.example.com` |
| CAA | 인증서 발급 허용 CA | `example.com → "letsencrypt.org"` |
| ALIAS/ANAME | CNAME의 루트 도메인 확장 | Cloudflare/Route53 전용 (표준 아님) |

### CNAME vs A 레코드
- **A 레코드**: IP를 직접 지정. 루트 도메인(`@`)에 사용 가능
- **CNAME**: 다른 도메인을 가리킴. 루트 도메인에 사용 불가 (RFC 1034)
- **CNAME Flattening**: Cloudflare 등에서 루트 도메인에 CNAME처럼 사용 — Auth NS가 내부적으로 A 레코드로 변환해서 응답

### SOA 레코드 상세
```
example.com. IN SOA ns1.example.com. admin.example.com. (
    2024010101  ; Serial (YYYYMMDDNN)
    3600        ; Refresh (Secondary NS가 Primary에 변경 확인 주기)
    900         ; Retry (Refresh 실패 시 재시도 간격)
    604800      ; Expire (Secondary NS가 Primary 연결 못하면 데이터 폐기 시간)
    300         ; Minimum TTL (Negative Caching TTL)
)
```

---

## TTL (Time To Live)

- DNS 응답이 캐시에 저장되는 시간 (초 단위)
- **낮은 TTL**: 변경 사항이 빠르게 반영 (배포/페일오버 시 유리), DNS 쿼리 부하 증가
- **높은 TTL**: 캐시 효율 좋음, 변경 반영이 느림

### Negative Caching (RFC 2308)
- NXDOMAIN 응답도 캐싱 → 존재하지 않는 도메인 반복 쿼리 방지
- TTL = SOA의 Minimum TTL 값
- 오타 도메인 쿼리가 캐싱되면 SOA TTL 만료 전까지 못 찾음

### 배포/마이그레이션 전략
```
마이그레이션 전: TTL을 300초(5분)로 낮춤 → 기존 TTL만큼 대기
마이그레이션 실행: DNS 레코드 변경
완료 후: TTL을 다시 높임 (3600초 등)
```

---

## DNS 고가용성 & 성능

### Anycast DNS
- Root NS 13개 주소 → 실제로는 전 세계 수백 개 서버가 동일 IP를 Anycast로 광고
- 클라이언트는 라우팅상 가장 가까운 서버와 통신
- Cloudflare 1.1.1.1, Google 8.8.8.8 모두 Anycast

### GeoDNS (지리적 라우팅)
- Recursive Resolver의 IP를 기반으로 가장 가까운 서버 IP 반환
- Route 53 Latency-based / Geolocation 라우팅이 대표적
- EDNS Client Subnet (ECS): Resolver가 클라이언트 서브넷 정보를 Auth NS에 전달 → 더 정확한 지리 라우팅

```
서울 사용자 → 8.8.8.8 → Auth NS (ECS: 203.0.0.0/24 포함)
                              → 서울 리전 IP 반환
```

### DNS 기반 로드밸런싱
- **Round Robin DNS**: 같은 도메인에 여러 A 레코드 → 순서 로테이션
  - 단점: 클라이언트 캐싱으로 분산 불균등, 서버 장애 감지 불가
- **Health Check 연동**: Route 53 Health Check + Failover → 장애 서버 IP 자동 제거
- **Weighted Routing**: 트래픽 비율 지정 (카나리 배포에 활용)

---

## DNS 캐싱 계층

```
[1] 브라우저 내부 캐시 (Chrome: chrome://net-internals/#dns)
[2] OS DNS 캐시(nscd, systemd-resolved)
[3] /etc/hosts (DNS보다 우선)
[4] Recursive Resolver 캐시 (ISP or 8.8.8.8)
[5] Auth NS (캐시 없음, 항상 권한 있는 응답)
```

- 캐시 계층별 TTL이 독립적으로 동작
- Recursive Resolver 캐시 히트 → Root/TLD/Auth NS 쿼리 생략

---

## DNS 보안

### DNS Spoofing / Cache Poisoning
- 가짜 DNS 응답을 Resolver 캐시에 주입 → 사용자를 악성 사이트로 유도
- **Kaminsky Attack (2008)**: Transaction ID + Source Port 조합 무작위 대입으로 Cache Poisoning
- **방어**: DNSSEC, Source Port Randomization, 0x20 encoding

### DNSSEC (DNS Security Extensions)
```
Root Zone Key (KSK) → TLD Zone Key → example.com Zone Key
각 단계에서 RRSIG로 서명 → 신뢰 체인 (Chain of Trust)
```
- **RRset**: 같은 타입의 레코드 집합
- **RRSIG**: RRset에 대한 디지털 서명
- **DNSKEY**: Zone의 공개키 (ZSK + KSK)
- **DS Record**: 상위 Zone에 하위 Zone의 KSK 해시 등록 → 신뢰 연결
- **단점**: 응답 크기 증가, 키 롤오버 복잡, 증폭 공격에 악용 가능

### DNS over HTTPS (DoH) / DNS over TLS (DoT)
| | DoH | DoT |
|--|-----|-----|
| 포트 | 443 | 853 |
| 프로토콜 | HTTPS | TLS |
| 특징 | 일반 웹 트래픽과 구분 불가 | 별도 포트로 방화벽 차단 가능 |
| 지원 | Chrome, Firefox 기본 지원 | Android, systemd-resolved |

### DNS 증폭 공격 (DDoS)
- 작은 DNS 쿼리 → 큰 DNS 응답 (DNSSEC 포함 시 더 큼)
- UDP Spoofing으로 피해자 IP로 대량 응답 유도
- 방어: Response Rate Limiting (RRL), BCP38 (Ingress Filtering)

### Split-horizon DNS
- 내부/외부 네트워크에서 같은 도메인에 다른 IP 반환
- 내부: 프라이빗 IP, 외부: 퍼블릭 IP
- Kubernetes CoreDNS, AWS Route 53 Private Hosted Zone이 대표적 구현

---

## Kubernetes DNS (CoreDNS)

```
Pod → CoreDNS (kube-dns Service: 10.96.0.10)
    → 클러스터 내부: <service>.<namespace>.svc.cluster.local
    → 외부 도메인: 업스트림 DNS로 포워딩
```

### /etc/resolv.conf (Pod 내부)
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- `ndots:5`: 점이 5개 미만이면 search 도메인 먼저 시도 → 불필요한 쿼리 다수 발생
- `redis` → `redis.default.svc.cluster.local` 로 자동 확장
- Headless Service: ClusterIP 없이 각 Pod IP를 직접 DNS로 반환

### CoreDNS 설정 (Corefile)
```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . 8.8.8.8          # 외부 쿼리 업스트림
    cache 30
    loop
    reload
    loadbalance
}
```

---

## DNS 진단 명령어

```bash
# 기본 조회
dig example.com
dig example.com A          # A 레코드
dig example.com MX         # MX 레코드
dig -x 8.8.8.8             # 역방향 조회 (PTR)

# 특정 네임서버에 질의
dig @8.8.8.8 example.com

# 전체 조회 과정 추적 (Root → TLD → Auth)
dig +trace example.com

# DNSSEC 검증 확인
dig +dnssec example.com

# 응답 지연 측정
dig +stats example.com

# 빠른 확인
nslookup example.com
host example.com

# 로컬 DNS 캐시 초기화 (macOS)
sudo killall -HUP mDNSResponder

# systemd-resolved 상태 확인 (Linux)
resolvectl status
resolvectl query example.com
```

---

## 실무 포인트

- **`/etc/resolv.conf`**: 시스템 DNS 서버 설정. `nameserver`, `search` 도메인
- **`/etc/hosts`**: DNS보다 우선 적용. 로컬 개발/테스트에 활용
- **DNS Propagation**: 전 세계 DNS 서버에 변경 사항이 전파되는 데 TTL만큼 소요
- **Route 53 Health Check**: DNS 레벨 장애 감지 → 자동 페일오버 (TTL 60초 이하 권장)
- **Negative Caching**: 존재하지 않는 도메인도 캐싱됨 (NXDOMAIN). SOA의 Minimum TTL 기준
- **SERVFAIL**: 네임서버 오류 (설정 문제, 네트워크 차단, DNSSEC 검증 실패)
- **NXDOMAIN**: 도메인이 존재하지 않음
- **K8s ndots 이슈**: `ndots:5`로 인해 클러스터 외부 도메인 조회 시 불필요한 search 도메인 시도 → latency 증가. 완전한 FQDN 사용 (`example.com.`) 또는 ndots 낮추기로 해결
- **ECS (EDNS Client Subnet)**: 프라이버시 vs 지리 라우팅 정확도 트레이드오프. Cloudflare는 ECS 미지원
