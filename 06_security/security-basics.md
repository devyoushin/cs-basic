# 보안 기초 (DevOps/SRE)

## 인증 (Authentication) vs 인가 (Authorization)

- **인증 (AuthN)**: "당신이 누구인가?" — 신원 확인
- **인가 (AuthZ)**: "당신이 무엇을 할 수 있는가?" — 권한 확인

---

## TLS/SSL 인증서

```bash
# 자가 서명 인증서 생성 (개발용)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# 인증서 만료일 확인 (운영 필수 모니터링)
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -enddate

# Let's Encrypt (무료 자동 갱신)
certbot --nginx -d example.com -d www.example.com
```

### 인증서 갱신 자동화
```bash
# crontab
0 0 1 * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

---

## 시크릿 관리

### 절대 하면 안 되는 것
```bash
# 코드에 시크릿 하드코딩 금지!
DB_PASSWORD="super_secret_password"  # NO!
```

### 올바른 방법

**환경 변수**
```bash
export DB_PASSWORD=$(aws ssm get-parameter --name /prod/db/password --with-decryption --query Parameter.Value --output text)
```

**AWS Secrets Manager**
```python
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

db_creds = get_secret('prod/database/credentials')
```

**HashiCorp Vault**
```bash
# 시크릿 저장
vault kv put secret/prod/db password=mysecret

# 시크릿 읽기
vault kv get -field=password secret/prod/db
```

**쿠버네티스 Secret**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: bXlzZWNyZXQ=  # base64 인코딩
```
> 주의: K8s Secret은 기본적으로 etcd에 평문 저장. KMS 암호화 활성화 권장.

---

## 최소 권한 원칙 (Least Privilege)

```json
// Bad: 모든 권한
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// Good: 필요한 권한만
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

### IAM 모범 사례
- Root 계정 사용 금지 (MFA 필수)
- 서비스에는 IAM Role 사용 (액세스 키 금지)
- 권한은 최소한으로, 필요할 때 추가
- 정기적으로 미사용 권한/키 정리

---

## 네트워크 보안

### 보안 그룹 (AWS Security Group)
```
Inbound:
- 22 (SSH): 특정 IP만 허용 (0.0.0.0/0 금지!)
- 80/443: 0.0.0.0/0 (퍼블릭 웹)
- 3306: 앱서버 보안그룹만 허용

Outbound:
- 기본적으로 모두 허용, 필요시 제한
```

### VPC 격리
```
Public Subnet:  로드밸런서, Bastion Host
Private Subnet: 앱서버, DB서버 (인터넷 직접 접근 불가)
NAT Gateway:    Private → Internet 단방향 통신
```

### Bastion Host (점프 서버)
```
로컬 → [인터넷] → Bastion Host (Public) → 내부 서버 (Private)
```

SSH ProxyJump 설정:
```bash
# ~/.ssh/config
Host bastion
  HostName 1.2.3.4
  User ec2-user
  IdentityFile ~/.ssh/key.pem

Host internal-server
  HostName 10.0.1.100
  User ec2-user
  ProxyJump bastion
```

---

## 암호화

### 대칭키 vs 비대칭키

| 구분 | 대칭키 | 비대칭키 |
|------|--------|---------|
| 키 | 하나의 키로 암호화/복호화 | 공개키(암호화) + 개인키(복호화) |
| 속도 | 빠름 | 느림 |
| 키 배포 | 어려움 (키를 안전하게 전달해야) | 공개키는 자유롭게 배포 |
| 알고리즘 | AES-256 | RSA, ECC |
| 사용 | 데이터 암호화 | 키 교환, 인증서, SSH |

TLS는 비대칭키로 세션키(대칭키)를 교환한 후 대칭키로 통신.

### 해시
- 단방향 (복호화 불가)
- **bcrypt/Argon2**: 비밀번호 해시 (느린 해시, 솔트 포함) ← 비밀번호에 사용
- **SHA-256**: 빠른 해시, 파일 무결성 검증
- **MD5**: 취약, 사용 금지

---

## 컨테이너 보안

```dockerfile
# Bad: root로 실행
FROM ubuntu
RUN apt-get install -y myapp
CMD ["myapp"]

# Good: non-root 사용자
FROM ubuntu
RUN useradd -r -s /bin/false appuser
RUN apt-get install -y myapp
USER appuser
CMD ["myapp"]
```

```yaml
# Kubernetes Security Context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

---

## OWASP Top 10 (인프라 관점)

1. **Broken Access Control**: IAM 최소 권한, RBAC 구현
2. **Cryptographic Failures**: TLS 강제, 취약한 암호화 금지
3. **Injection**: 파라미터화 쿼리, WAF
4. **Security Misconfiguration**: 기본 자격증명 변경, 불필요한 포트 닫기
5. **Vulnerable Components**: 의존성 정기 업데이트, Snyk/Trivy
6. **Logging Failures**: 감사 로그, 이상 행위 탐지
7. **SSRF**: 내부 메타데이터 서비스 접근 차단

---

## 실무 포인트

- **시크릿 스캔**: `git-secrets`, `truffleHog`로 코드에 시크릿 포함 방지 (CI 파이프라인 통합)
- **컨테이너 이미지 스캔**: `trivy`, `Clair`로 CVE 취약점 스캔 (CI 필수)
- **AWS IMDSv2**: 메타데이터 서비스 SSRF 공격 방지. IMDSv2 강제 설정
- **WAF (Web Application Firewall)**: SQL 인젝션, XSS 차단. AWS WAF, Cloudflare
- **Security Group vs NACL**: SG는 Stateful(자동 응답 허용), NACL은 Stateless(응답도 명시)
- **CloudTrail + Config**: AWS API 호출 로그 + 리소스 변경 이력. 감사 필수
- **Key Rotation**: 접근 키는 90일마다 교체. Secrets Manager 자동 교체 활용
