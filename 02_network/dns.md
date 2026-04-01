# DNS (Domain Name System)

## DNS 조회 흐름

```
브라우저 → OS 캐시 → /etc/hosts → Local DNS Resolver(ISP)
          → Root NS → TLD NS (.com) → Authoritative NS
          → 응답 캐싱 (TTL 만큼)
```

### 상세 조회 과정
1. `www.example.com` 요청
2. Root NS에 `.com` NS 주소 질의
3. `.com` TLD NS에 `example.com` NS 주소 질의
4. `example.com` Authoritative NS에 `www.example.com` IP 질의
5. IP 반환 + TTL 기간 캐싱

---

## DNS 레코드 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| A | 도메인 → IPv4 | `api.example.com → 1.2.3.4` |
| AAAA | 도메인 → IPv6 | `api.example.com → ::1` |
| CNAME | 도메인 → 도메인 (별칭) | `www → example.com` |
| MX | 메일 서버 | `example.com → mail.example.com` |
| TXT | 임의 텍스트 | SPF, DKIM, 도메인 인증 |
| NS | 네임서버 지정 | `example.com → ns1.cloudflare.com` |
| PTR | IP → 도메인 (역방향) | `1.2.3.4 → api.example.com` |
| SOA | Zone의 권한 정보 | 시리얼 번호, 갱신 주기 |
| SRV | 서비스 위치 | `_http._tcp.example.com` |

### CNAME vs A 레코드
- **A 레코드**: IP를 직접 지정. 루트 도메인(`@`)에 사용 가능
- **CNAME**: 다른 도메인을 가리킴. 루트 도메인에 사용 불가 (일부 CDN은 ALIAS/ANAME으로 우회)
- **CNAME Flattening**: Cloudflare 등에서 루트 도메인에 CNAME처럼 사용하는 기능

---

## TTL (Time To Live)

- DNS 응답이 캐시에 저장되는 시간 (초 단위)
- **낮은 TTL**: 변경 사항이 빠르게 반영 (배포/페일오버 시 유리), DNS 쿼리 부하 증가
- **높은 TTL**: 캐시 효율 좋음, 변경 반영이 느림

### 배포/마이그레이션 전략
```
마이그레이션 전: TTL을 300초(5분)로 낮춤
마이그레이션 후: DNS 레코드 변경
이후: TTL을 다시 높임 (3600초 등)
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

# 전체 조회 과정 추적
dig +trace example.com

# 빠른 확인
nslookup example.com
host example.com

# 로컬 DNS 캐시 확인 (macOS)
sudo killall -HUP mDNSResponder  # DNS 캐시 초기화
```

---

## DNS 보안

### DNS Spoofing/Cache Poisoning
- 가짜 DNS 응답으로 사용자를 악성 사이트로 유도
- **방어**: DNSSEC (응답에 디지털 서명)

### DNSSEC
- DNS 응답의 무결성 검증 (위변조 방지)
- 공개키 암호화 사용

### DNS over HTTPS (DoH) / DNS over TLS (DoT)
- DNS 쿼리를 암호화 → ISP의 DNS 감청 방지
- Cloudflare(1.1.1.1), Google(8.8.8.8) 지원

### Split-horizon DNS
- 내부 네트워크와 외부에서 같은 도메인에 대해 다른 IP 반환
- 내부: 프라이빗 IP, 외부: 퍼블릭 IP
- 온프레미스 + 클라우드 하이브리드 환경에서 일반적

---

## 실무 포인트

- **`/etc/resolv.conf`**: 시스템 DNS 서버 설정. `nameserver`, `search` 도메인
- **`/etc/hosts`**: DNS보다 우선 적용. 로컬 개발/테스트에 활용
- **쿠버네티스 CoreDNS**: 클러스터 내부 서비스 디스커버리 담당 (`svc.cluster.local`)
- **DNS Propagation**: 전 세계 DNS 서버에 변경 사항이 전파되는 데 TTL만큼 소요
- **Route 53 Health Check**: DNS 레벨 장애 감지 → 자동 페일오버
- **Negative Caching**: 존재하지 않는 도메인도 캐싱됨 (NXDOMAIN). SOA의 TTL 기준
- **SERVFAIL**: 네임서버 오류 (설정 문제, 네트워크 차단 등)
- **NXDOMAIN**: 도메인이 존재하지 않음
