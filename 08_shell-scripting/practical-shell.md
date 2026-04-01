# Shell Scripting 실전

## 스크립트 기본 구조

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e: 명령어 실패 시 즉시 종료
# -u: 정의되지 않은 변수 사용 시 에러
# -o pipefail: 파이프 중간 실패도 감지

IFS=$'\n\t'  # 단어 분리 기준을 공백 대신 개행/탭으로 제한
```

---

## sed 실전

### 기본 문법
```bash
sed 's/찾을패턴/바꿀내용/플래그' 파일
# 플래그: g(전체), i(대소문자 무시), p(출력), d(삭제)
```

### 텍스트 치환
```bash
# 첫 번째 매칭만 치환
sed 's/foo/bar/' file.txt

# 줄 전체에서 모두 치환 (g 플래그)
sed 's/foo/bar/g' file.txt

# 원본 파일 직접 수정 (in-place)
sed -i 's/foo/bar/g' file.txt
sed -i.bak 's/foo/bar/g' file.txt  # .bak 백업 생성 후 수정

# 특정 줄 번호만 치환
sed '3s/foo/bar/' file.txt          # 3번째 줄만
sed '2,5s/foo/bar/' file.txt        # 2~5번째 줄
```

### 줄 삭제 & 추출
```bash
# 패턴 매칭 줄 삭제
sed '/^#/d' file.txt                # # 으로 시작하는 줄 삭제 (주석 제거)
sed '/^$/d' file.txt                # 빈 줄 삭제
sed '/^#/d; /^$/d' file.txt         # 주석 + 빈 줄 동시 삭제

# 특정 줄만 추출
sed -n '5,10p' file.txt             # 5~10번째 줄만 출력
sed -n '/START/,/END/p' file.txt    # START~END 구간 출력
```

### 줄 추가 & 삽입
```bash
# 특정 패턴 다음 줄에 추가 (a)
sed '/pattern/a\추가할 내용' file.txt

# 특정 패턴 이전 줄에 삽입 (i)
sed '/pattern/i\삽입할 내용' file.txt

# 설정 파일에 값 추가 (없을 때만)
grep -q 'vm.swappiness' /etc/sysctl.conf \
  || sed -i '/^$/d; $a\vm.swappiness=10' /etc/sysctl.conf
```

### 실전 활용
```bash
# nginx 설정에서 특정 서버 블록 비활성화
sed -i 's/listen 80/# listen 80/' /etc/nginx/sites-available/default

# 환경 변수 파일에서 값 변경
sed -i "s/^DB_HOST=.*/DB_HOST=newhost/" .env

# 멀티라인: 여러 명령 한 번에
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# 구분자 변경 (경로 치환 시 / 대신 | 사용)
sed 's|/old/path|/new/path|g' config.txt
```

---

## awk 실전

### 기본 구조
```
awk 'BEGIN{초기화} /패턴/{액션} END{마무리}' 파일

내장 변수:
  $0      전체 줄
  $1,$2.. 공백 기준 필드
  NR      현재 줄 번호 (Number of Records)
  NF      현재 줄의 필드 수 (Number of Fields)
  FS      입력 구분자 (기본: 공백)
  OFS     출력 구분자 (기본: 공백)
  RS      레코드 구분자 (기본: 개행)
```

### 필드 추출 & 가공
```bash
# 특정 컬럼만 출력
awk '{print $1, $3}' file.txt

# 구분자 지정 (-F)
awk -F: '{print $1, $3}' /etc/passwd      # 사용자명, UID 출력
awk -F',' '{print $1}' data.csv           # CSV 첫 번째 컬럼

# 출력 구분자 변경
awk -F: 'OFS="\t" {print $1, $3, $6}' /etc/passwd

# 조건부 출력
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd   # UID 1000 이상 사용자
awk 'NF > 3' file.txt                              # 필드가 3개 초과인 줄만
awk '$0 !~ /^#/ && NF' config.txt                  # 주석/빈 줄 제외
```

### 집계 & 통계
```bash
# 합계
awk '{sum += $1} END {print sum}' numbers.txt

# 평균
awk '{sum += $1; count++} END {print sum/count}' numbers.txt

# 최대/최소
awk 'NR==1 || $1>max {max=$1} END {print max}' numbers.txt

# nginx 로그에서 IP별 요청 수
awk '{ip[$1]++} END {for (k in ip) print ip[k], k}' access.log \
  | sort -rn | head -10

# HTTP 상태코드별 집계
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 응답 크기 합계 (바이트)
awk '{sum += $10} END {printf "Total: %.2f MB\n", sum/1024/1024}' access.log
```

### 조건문 & 반복문
```bash
# if/else
awk '{
  if ($3 > 100) print $1, "HIGH"
  else if ($3 > 50) print $1, "MED"
  else print $1, "LOW"
}' data.txt

# for 반복
awk '{for(i=1; i<=NF; i++) printf "%s ", $i; print ""}' file.txt

# 패턴 범위 (START ~ END 구간)
awk '/BEGIN_MARKER/,/END_MARKER/' file.txt
```

### 실전 활용
```bash
# 중복 줄 제거 (순서 유지, uniq와 달리 인접하지 않아도 됨)
awk '!seen[$0]++' file.txt

# 특정 컬럼 기준 중복 제거
awk -F, '!seen[$1]++' data.csv

# 두 파일 조인 (SQL JOIN처럼)
awk -F: 'NR==FNR {a[$1]=$2; next} $1 in a {print $1, a[$1], $2}' \
  file1.txt file2.txt

# CSV를 보기 좋게 정렬 출력
awk -F, '{printf "%-20s %-10s %-10s\n", $1, $2, $3}' data.csv

# 프로세스별 메모리 합산 (ps 출력 가공)
ps aux | awk 'NR>1 {mem[$11]+=$6} END {for(p in mem) print mem[p], p}' \
  | sort -rn | head -10

# 로그에서 응답시간 p95 계산
awk '{print $NF}' access.log | sort -n \
  | awk 'BEGIN{c=0} {a[c++]=$1} END{print a[int(c*0.95)]}'
```

---

## grep 실전

```bash
# 기본 패턴
grep 'pattern' file.txt
grep -i 'pattern' file.txt          # 대소문자 무시
grep -v 'pattern' file.txt          # 역매칭 (패턴 없는 줄)
grep -r 'pattern' ./dir/            # 재귀 탐색
grep -l 'pattern' *.txt             # 매칭된 파일명만 출력
grep -c 'pattern' file.txt          # 매칭된 줄 수

# 확장 정규식 (-E = egrep)
grep -E 'error|warn|crit' /var/log/syslog
grep -E '^[0-9]{4}-[0-9]{2}' logfile

# 컨텍스트 출력
grep -A 3 'ERROR' app.log           # 매칭 후 3줄
grep -B 3 'ERROR' app.log           # 매칭 전 3줄
grep -C 3 'ERROR' app.log           # 매칭 전후 3줄

# 실전
grep -rn 'TODO\|FIXME' ./src/       # 소스에서 TODO/FIXME 찾기
grep -P '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}' file  # IP 주소 추출 (Perl 정규식)
grep -oP '"status":\K\d+' response.json  # JSON에서 status 값만 추출
```

---

## 파이프 조합 실전 패턴

### 로그 분석
```bash
# 최근 1시간 에러 로그 집계
awk -v date="$(date -d '1 hour ago' '+%Y-%m-%dT%H')" \
  '$0 ~ date && /ERROR/' app.log \
  | awk '{print $5}' | sort | uniq -c | sort -rn

# 느린 쿼리 로그에서 상위 10개 추출
grep 'Query_time' slow.log \
  | awk '{print $3}' \
  | sort -rn | head -10

# 접속 IP 국가별 집계 (geoip 설치 필요)
awk '{print $1}' access.log | sort -u \
  | xargs -I{} geoiplookup {} \
  | awk -F: '{print $2}' | sort | uniq -c | sort -rn
```

### 시스템 모니터링
```bash
# CPU 사용률 상위 5개 프로세스 (1초 간격 3회)
for i in {1..3}; do
  ps aux --sort=-%cpu | awk 'NR==1 || NR<=6' | awk '{printf "%-10s %5s%%\n", $11, $3}'
  sleep 1
done

# 디스크 사용률 80% 이상인 파티션
df -h | awk 'NR>1 && int($5) >= 80 {print $0}'

# 포트별 연결 수 집계
ss -tn | awk 'NR>1 {split($5,a,":"); port[a[2]]++} END {for(p in port) print port[p], p}' \
  | sort -rn

# 메모리 사용량 상위 프로세스
ps aux | awk 'NR>1 {print $6/1024, $11}' | sort -rn | head -10 \
  | awk '{printf "%8.1f MB  %s\n", $1, $2}'
```

### 파일 처리
```bash
# 여러 파일의 특정 패턴 줄만 모아서 새 파일로
find /var/log -name "*.log" -newer /tmp/last_check \
  | xargs grep -h 'ERROR' > /tmp/errors_$(date +%Y%m%d).txt

# CSV 파일 특정 컬럼 기준 정렬 후 중복 제거
sort -t',' -k2,2 data.csv | awk -F, '!seen[$2]++'

# 설정 파일에서 활성화된 항목만
grep -v '^\s*#' config.conf | grep -v '^\s*$' | sed 's/\s*=\s*/=/'
```

---

## 함수 & 재사용 가능한 스크립트 패턴

```bash
#!/usr/bin/env bash
set -euo pipefail

# 로깅 함수
log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] INFO  $*"; }
warn() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] WARN  $*" >&2; }
die()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR $*" >&2; exit 1; }

# 의존성 체크
require() {
  for cmd in "$@"; do
    command -v "$cmd" &>/dev/null || die "'$cmd' 명령어가 필요합니다"
  done
}

# 재시도 함수
retry() {
  local max=$1; shift
  local delay=5
  local attempt=1
  until "$@"; do
    (( attempt++ > max )) && die "최대 재시도($max) 초과: $*"
    warn "재시도 $attempt/$max (${delay}초 후)..."
    sleep $delay
    (( delay *= 2 ))  # exponential backoff
  done
}

# 락 파일 (중복 실행 방지)
LOCKFILE="/tmp/$(basename "$0").lock"
exec 9>"$LOCKFILE"
flock -n 9 || die "이미 실행 중입니다"
trap 'rm -f "$LOCKFILE"' EXIT

# 사용 예시
require curl jq aws
retry 3 curl -sf https://api.example.com/health
log "스크립트 완료"
```

---

## xargs & 병렬 처리

```bash
# 기본 xargs
find . -name "*.log" | xargs rm -f
cat urls.txt | xargs -I{} curl -O {}

# 병렬 실행 (-P: 병렬 수)
cat servers.txt | xargs -P 10 -I{} ssh {} 'uptime'

# 파일 이름에 공백 대비 (-0 옵션)
find . -name "*.txt" -print0 | xargs -0 grep 'pattern'

# GNU Parallel (xargs보다 강력)
cat hosts.txt | parallel -j 20 'ssh {} "df -h | grep -v tmpfs"'
parallel -j 4 convert {} {.}.jpg ::: *.png  # 이미지 변환 병렬화
```

---

## 실무 포인트

- **`set -euo pipefail`은 모든 스크립트 첫 줄에** — 없으면 오류를 무시하고 계속 실행해 데이터 손실 가능
- **`awk`가 `grep | cut | sort`보다 빠름** — 파이프를 여러 번 연결하면 fork 비용 발생. awk 하나로 통합이 효율적
- **큰 파일은 `sed -n`** — 기본 sed는 전체를 출력하므로 필요한 부분만 `-n`으로 추출
- **로그 분석 시 `LC_ALL=C`** — 로케일 처리를 건너뛰어 정렬/비교 속도 대폭 향상 (`LC_ALL=C sort -rn`)
- **`awk NR==FNR` 패턴** — 두 파일을 메모리에서 조인할 때 핵심 관용구. 첫 번째 파일 처리 시 NR==FNR
- **디버깅**: `bash -x script.sh` — 모든 명령어와 인수를 실행 전에 출력. `set -x`/`set +x`로 구간 지정 가능
