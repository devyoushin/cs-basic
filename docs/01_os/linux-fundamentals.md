# Linux 기초

## 파일시스템 구조

```
/
├── bin/    # 기본 명령어 (ls, cp, mv...)
├── etc/    # 설정 파일
├── var/    # 가변 데이터 (로그, 스풀)
├── tmp/    # 임시 파일 (재부팅 시 삭제)
├── proc/   # 가상 FS - 커널/프로세스 정보
├── sys/    # 가상 FS - 장치/드라이버 정보
├── dev/    # 장치 파일
├── home/   # 사용자 홈 디렉토리
└── usr/    # 사용자 프로그램
```

## 핵심 명령어

### 프로세스 관리
```bash
ps aux                    # 전체 프로세스 목록
ps aux | grep nginx       # 특정 프로세스 검색
top / htop                # 실시간 프로세스 모니터링
kill -9 <PID>             # 강제 종료 (SIGKILL)
kill -15 <PID>            # 정상 종료 (SIGTERM) — 권장
lsof -i :8080             # 특정 포트를 사용하는 프로세스
pgrep -f "process_name"   # 프로세스 이름으로 PID 검색
```

### 시스템 리소스 확인
```bash
free -h                   # 메모리 사용량
df -h                     # 디스크 사용량
du -sh /var/log/*         # 디렉토리별 디스크 사용량
iostat -x 1               # 디스크 I/O 통계
vmstat 1                  # 가상 메모리 통계
uptime                    # 시스템 부하 (load average)
```

### 네트워크
```bash
ss -tulnp                 # 열린 포트 및 소켓 (netstat 대체)
netstat -tulnp            # 열린 포트 목록 (구버전)
curl -v https://example.com  # HTTP 요청 + 상세 출력
wget -O /dev/null http://example.com  # 다운로드 테스트
tcpdump -i eth0 port 80   # 패킷 캡처
iperf3 -s / -c <host>     # 네트워크 대역폭 테스트
```

### 로그 분석
```bash
tail -f /var/log/syslog          # 실시간 로그 확인
journalctl -u nginx -f           # systemd 서비스 로그
journalctl --since "1 hour ago"  # 시간 기준 로그 필터
grep -r "ERROR" /var/log/        # 재귀 에러 검색
awk '{print $1}' access.log | sort | uniq -c | sort -rn  # IP별 접근 횟수
```

### 고급 진단 도구

```bash
# strace: 시스템 콜 추적
strace -p <PID>                   # 실행 중인 프로세스의 syscall
strace -c ls /tmp                 # syscall 통계 요약
strace -e trace=open,read <cmd>   # 특정 syscall만 필터
strace -T -p <PID>                # syscall별 소요 시간 포함

# lsof: 열린 파일/소켓 조회
lsof -p <PID>                     # 특정 프로세스가 연 파일
lsof -i :8080                     # 특정 포트 사용 프로세스
lsof -i TCP -n -P                 # 모든 TCP 소켓
lsof -u root | wc -l              # root가 연 FD 수
lsof +D /var/log                  # 특정 디렉토리 사용 프로세스

# vmstat: 가상 메모리 & CPU 통계
vmstat 1 5                        # 1초 간격 5회
# r=실행큐, b=블로킹, si/so=스왑인/아웃, cs=컨텍스트스위칭, us/sy/id=CPU
vmstat -s                         # 메모리 요약

# iostat: 디스크 I/O 통계
iostat -x 1                       # 확장 통계 1초 간격
# %util=장치 사용률, await=I/O 대기시간, r/s w/s=읽기/쓰기 IOPS
iostat -d -h 1                    # 사람이 읽기 쉬운 형식

# sar: 시스템 활동 기록 & 조회
sar -u 1 5                        # CPU 1초 간격
sar -n DEV 1                      # 네트워크 인터페이스 통계
sar -r 1                          # 메모리 통계
sar -d 1                          # 디스크 I/O

# perf: 성능 프로파일링
perf top                          # 실시간 CPU 핫스팟
perf stat <cmd>                   # 명령어 실행 통계 (cache miss, branch miss 등)
perf record -g ./app && perf report  # 플레임 그래프용 데이터 수집
```

---

## 파일 권한

```
-rwxr-xr-- 1 user group 4096 Jan 1 00:00 file
 |||||||└── other: r--  (4)
 ||||└───── group: r-x  (5)
 |└──────── owner: rwx  (7)
 └───────── 파일 타입 (- 파일, d 디렉토리, l 심링크)
```

```bash
chmod 755 file      # rwxr-xr-x
chmod +x script.sh  # 실행 권한 추가
chown user:group file
```

---

## 시그널

| 시그널 | 번호 | 의미 |
|--------|------|------|
| SIGHUP | 1 | 재시작 (설정 재로드) |
| SIGINT | 2 | 인터럽트 (Ctrl+C) |
| SIGKILL | 9 | 강제 종료 (무시 불가) |
| SIGTERM | 15 | 정상 종료 요청 (기본값) |
| SIGUSR1 | 10 | 사용자 정의 (nginx: 로그 rotate) |

---

## /proc 활용

```bash
cat /proc/cpuinfo          # CPU 정보
cat /proc/meminfo          # 메모리 상세 정보
cat /proc/<PID>/limits     # 프로세스 리소스 제한
cat /proc/<PID>/fd/ | wc -l  # 파일 디스크립터 수
cat /proc/sys/net/core/somaxconn  # 소켓 백로그 최대값
```

---

## systemd

```bash
systemctl status nginx       # 서비스 상태
systemctl start/stop/restart nginx
systemctl enable nginx       # 부팅 시 자동 시작
systemctl daemon-reload      # 유닛 파일 변경 후 재로드
systemctl list-units --failed  # 실패한 서비스 목록
```

### 서비스 유닛 파일 예시
```ini
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
User=appuser
ExecStart=/usr/bin/myapp
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

---

## 실무 포인트

- **Load Average** = 1분/5분/15분 평균 실행 대기 프로세스 수. CPU 코어 수와 비교 (4코어면 4.0이 100% 부하)
- **OOM Killer** = 메모리 부족 시 커널이 프로세스 강제 종료. `/var/log/syslog`에서 `Out of memory` 확인
- **파일 디스크립터 한도** = `ulimit -n` 확인. 고트래픽 서버는 65536 이상 설정 필요
- **inode 고갈** = `df -i`로 확인. 파일 수가 너무 많으면 디스크 공간이 있어도 파일 생성 불가
- **zombie 프로세스** = 부모 프로세스가 wait()를 안 한 경우. `ps aux | awk '$8=="Z"'`로 확인
- **strace 장애 분석**: `strace -c -p <PID>`로 syscall 통계 → 특정 syscall 비중 높으면 I/O 병목 또는 과도한 커널 전환
- **iostat %util 100%**: 디스크 포화. `await` 값이 높으면 I/O 대기 심각. SSD 교체 또는 I/O 패턴 최적화 필요
- **vmstat cs 높음**: 컨텍스트 스위칭 과다. 스레드 수 과다 또는 잦은 sleep/wakeup 패턴 의심
