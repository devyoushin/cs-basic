# CI/CD & 배포 전략

## CI/CD 파이프라인 구조

```
개발자 push
    ↓
CI (Continuous Integration)
    ├── 코드 빌드 (compile, lint)
    ├── 단위 테스트 / 통합 테스트
    ├── 정적 분석 (SAST, 취약점 스캔)
    ├── 컨테이너 이미지 빌드
    └── 이미지 레지스트리 푸시 (ECR, GCR, Harbor)
         ↓
CD (Continuous Delivery/Deployment)
    ├── 스테이징 배포 → 자동 검증
    ├── 승인 게이트 (수동 or 자동)
    └── 프로덕션 배포
```

### CI에서 중요한 원칙

**Fast Feedback Loop**: 문제를 빠르게 발견할수록 수정 비용이 낮음.
- 단위 테스트: 수 초 → 항상 실행
- 통합 테스트: 수 분 → PR 시 실행
- E2E 테스트: 수십 분 → 스테이징 배포 후 실행

**Hermetic Build (재현 가능한 빌드)**:
- 빌드 결과가 환경에 무관하게 동일 → 의존성 고정 (lock file)
- 빌드 시점의 외부 API 호출 금지
- Docker 이미지에 `latest` 태그 사용 금지 → SHA256 다이제스트로 고정

```dockerfile
# Bad
FROM node:latest

# Good
FROM node:20.11.0-alpine3.19@sha256:abc123...
```

---

## 배포 전략 비교

### 1. Recreate (재생성)

```
v1: [A][A][A] → 전체 중단 → v2: [B][B][B]
```

- 기존 버전 전체 중단 후 새 버전 전체 시작
- **장점**: 단순, 두 버전 동시 실행 없음 (DB 스키마 변경에 안전)
- **단점**: 다운타임 발생
- **적합**: 비용 절감이 중요한 배치 작업, 완전히 호환 안 되는 버전 교체

### 2. Rolling Update (롤링 업데이트)

```
[A][A][A] → [A][A][B] → [A][B][B] → [B][B][B]
```

- 구버전을 하나씩 교체
- **장점**: 다운타임 없음, 추가 인프라 불필요
- **단점**: 배포 중 두 버전이 동시 실행 → API 호환성 필수
- **K8s**: `maxSurge`, `maxUnavailable` 조정으로 속도/안전성 트레이드오프

**배포 중 두 버전 공존 문제**:
```
배포 진행 중:
  v1 Pod: /api/v1/users → {id, name, email}
  v2 Pod: /api/v1/users → {id, name, email, phone}   ← 필드 추가

→ 클라이언트 입장에서 같은 엔드포인트가 다른 응답을 반환
→ 신규 필드 추가는 Backward Compatible → OK
→ 필드 제거/타입 변경은 Breaking Change → 별도 처리 필요
```

### 3. Blue-Green 배포

```
현재: 트래픽 → [Blue: v1][v1][v1]
새버전 준비:   [Blue: v1]  +  [Green: v2] (트래픽 없음, 준비 중)
전환:         트래픽 → [Green: v2][v2][v2]
롤백:         트래픽 → [Blue: v1] (즉시)
```

- **장점**:
  - 무중단 + **즉시 롤백** (트래픽 전환만 되돌리면 됨)
  - Green 환경에서 충분히 검증 후 전환
  - 두 버전 동시 실행 시간 최소화 (전환 순간에만)
- **단점**:
  - **리소스 2배** 필요 (Blue + Green 동시 운영)
  - DB 스키마 변경 시 복잡 (두 버전이 같은 DB를 써야 함)
- **트래픽 전환 방법**:
  - DNS TTL 낮춘 후 DNS 변경 (수 분)
  - LB 타겟 그룹 전환 (수 초, AWS ALB weighted target group)
  - K8s Service selector 변경 (즉시)

### 4. Canary 배포

```
단계 1: v1 95% + v2 5%   (소수 사용자에게만)
단계 2: v1 80% + v2 20%  (오류율 정상 확인 후)
단계 3: v1 50% + v2 50%
단계 4: v1  0% + v2 100%
```

- **장점**:
  - 실제 트래픽으로 검증하며 점진적 확대
  - 문제 발생 시 영향 범위 최소화 (5%만 영향)
  - A/B 테스트와 결합 가능
- **단점**:
  - 두 버전 동시 실행 (호환성 필수)
  - 모니터링 체계 필요 (에러율, 레이턴시 자동 감지)
  - 완전 배포까지 시간 소요

**K8s Canary 구현 (Ingress 가중치)**:
```yaml
# nginx-ingress annotation 방식
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "20"   # 20% 트래픽
```

**자동 Canary (Argo Rollouts)**:
```yaml
steps:
- setWeight: 5
- pause: {duration: 10m}
- setWeight: 20
- analysis:              # 지표 자동 검증
    templates:
    - templateName: error-rate   # 에러율 < 1% 조건
- setWeight: 100
```

### 5. Feature Flag (기능 플래그)

배포와 기능 릴리스를 분리:
```python
# 코드에 플래그 내장
if feature_flags.is_enabled("new-checkout-flow", user_id):
    return new_checkout()
else:
    return old_checkout()
```

- 배포: 코드 전체 배포 (플래그는 비활성)
- 릴리스: 플래그 서버에서 특정 사용자 그룹에 활성화
- **장점**: 배포와 릴리스 독립 / 즉시 롤백 / 특정 사용자 타겟
- **도구**: LaunchDarkly, Unleash, AWS AppConfig

---

## Database Migration 전략

배포 시 가장 어려운 부분. **DB 스키마 변경은 롤백이 어렵다**.

### Expand-Contract 패턴

```
Before:  users(id, name, email)

Step 1 Expand: 새 컬럼 추가 (nullable, 기본값 있음)
  ALTER TABLE users ADD COLUMN phone VARCHAR(20);
  → 배포 후 v1/v2 모두 동작 (v1은 phone 무시, v2는 phone 사용)

Step 2 Migrate: 기존 데이터 채우기
  UPDATE users SET phone = '...' WHERE phone IS NULL;
  (대용량이면 배치로)

Step 3 Contract: v1 완전 제거 후
  ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;
  → 이제 NOT NULL 강제 가능
```

**컬럼 삭제**:
```
Step 1: 애플리케이션이 컬럼 참조 제거 후 배포 (DB는 아직 유지)
Step 2: 배포 완전 완료 후 컬럼 삭제
→ 롤백 시 컬럼이 있어도 앱이 무시하므로 안전
```

### Zero-downtime 마이그레이션 주의사항

```sql
-- Bad: 테이블 전체 락 (MySQL 5.7 이하)
ALTER TABLE orders ADD INDEX idx_user_id (user_id);
-- → 대용량 테이블에서 수 분~수 시간 서비스 중단

-- Good: Online DDL 또는 pt-online-schema-change
pt-online-schema-change \
  --alter="ADD INDEX idx_user_id (user_id)" \
  D=mydb,t=orders --execute
-- → 트리거 기반 변경, 서비스 중단 없음
```

---

## Git 브랜치 전략

### GitFlow

```
main ─────────────────────────────────────────────→
  ↑ merge (release)
develop ──────────────────────────────────────────→
  ↑ merge (feature)
feature/login ──────────────→ (merge to develop)
feature/payment ─────────────────────────→
  ↑ hotfix는 main에서 분기 후 main + develop에 merge
hotfix/cve-fix ─→ (main + develop)
```

- 명확한 릴리스 관리
- **단점**: 복잡한 브랜치 관리, 브랜치 분기가 길어지면 merge conflict 증가

### Trunk-Based Development (TBD)

```
main ─────────────────────────────────────────────→
  ↑ 짧은 브랜치 (1-2일)
feature/x ──→
feature/y ───→
feature/z ──→
```

- 모든 개발자가 `main`에 매일 통합
- Feature Flag로 미완성 기능 숨김
- CI/CD가 강력할 때 효과적 (빠른 빌드, 높은 테스트 커버리지)
- Google, Meta 등 대형 테크 기업이 사용

### 브랜치 전략 선택 기준

| 상황 | 권장 전략 |
|------|---------|
| 정해진 릴리스 주기 (2주마다 배포) | GitFlow |
| 지속적 배포 (하루 여러 번) | Trunk-Based |
| 작은 팀 + 빠른 이터레이션 | Trunk-Based |
| 외부 배포 제품 (패키지 소프트웨어) | GitFlow |

---

## Rollback 전략

### 빠른 롤백이 중요한 이유
- 장애 지속 시간 × 영향 사용자 수 = 비즈니스 손실
- MTTR(Mean Time To Recovery) 단축이 SRE 핵심 목표

### 배포 방식별 롤백 속도

| 방식 | 롤백 방법 | 소요 시간 |
|------|---------|---------|
| Blue-Green | LB 타겟 전환 | 수 초 |
| K8s Deployment | `kubectl rollout undo` | 수 분 |
| Canary | 가중치 0으로 변경 | 수 초~분 |
| Rolling Update | 이전 이미지로 재배포 | 수 분 |

```bash
# K8s 롤백
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3  # 특정 버전으로

# 롤아웃 히스토리 확인
kubectl rollout history deployment/my-app

# 배포 일시 중지/재개
kubectl rollout pause deployment/my-app   # 카나리처럼 중간에 멈춤
kubectl rollout resume deployment/my-app
```

---

## 실무 포인트

- **배포 전 체크리스트**: DB 마이그레이션 완료 여부, Feature Flag 상태, 모니터링 알람 준비, 롤백 절차 확인
- **배포 시간 선택**: 트래픽 낮은 시간대 (새벽) 또는 트래픽 안정된 시간대. 금요일 오후 배포 지양
- **헬스체크 엔드포인트**: `/health` (liveness), `/ready` (readiness) 반드시 구현. readiness는 DB 연결까지 확인
- **배포 자동 롤백 조건**: 에러율 > N%, p99 레이턴시 > N ms → 자동 이전 버전으로 되돌아가는 파이프라인
- **이미지 태그 전략**: `latest` 금지. `git-sha`, `v1.2.3`, `20260415-ab12cd` 형태로 추적 가능하게
- **GitOps (ArgoCD, Flux)**: Git이 프로덕션 상태의 진실 공급원. Git commit = 배포. 감사 로그 자동화
