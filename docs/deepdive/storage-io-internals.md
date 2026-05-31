# Storage & I/O 내부 동작 완전 분석 (Senior Level)

> 블록 I/O 스택, io_uring, NVMe 큐, 파일시스템 저널링, EBS/S3 클라우드 스토리지 내부

---

## 1. Linux I/O 스택 전체 구조

```
애플리케이션 (read/write syscall)
        │
        ▼
VFS (Virtual File System) Layer
  ← 파일시스템 추상화 (ext4, xfs, btrfs, tmpfs ...)
        │
        ▼
페이지 캐시 (Page Cache)
  ← 읽기: 캐시 히트 시 즉시 반환
  ← 쓰기: Dirty 페이지로 표시 후 반환 (Writeback)
        │
        ▼
파일시스템 (ext4, XFS, Btrfs)
  ← 논리 블록 → 물리 블록 매핑 (inode, extent tree)
  ← 저널링/WAL로 일관성 유지
        │
        ▼
블록 레이어 (Block Layer)
  ← I/O 스케줄러 (none, mq-deadline, kyber, bfq)
  ← I/O 병합, 재정렬
        │
        ▼
디바이스 드라이버 (NVMe, SCSI, virtio-blk)
        │
        ▼
물리 디바이스 (NVMe SSD, HDD, EBS)
```

### I/O 경로 상세

```bash
# I/O 스택 레이어별 레이턴시 측정
# 블록 I/O 레이턴시 히스토그램 (io_uring or block layer)
biolatency-bpfcc -d sda    # 장치별

# VFS 레이어 레이턴시
fileslower-bpfcc 1         # 1ms 이상 파일 작업

# 스케줄러 큐 대기 시간 포함
blklatency-bpfcc

# iostat으로 I/O 통계
iostat -xz 1
# await: I/O 완료까지 평균 시간 (큐 대기 + 처리)
# svctm: 실제 디바이스 처리 시간 (deprecated in newer kernels)
# %util: 디바이스가 I/O를 처리하는 시간 비율
# r_await, w_await: 읽기/쓰기 별 대기 시간
```

---

## 2. 페이지 캐시 (Page Cache)

### 읽기 경로

```
read() 시스템 콜:
  1. 파일 오프셋 → 페이지 캐시 조회 (페이지 단위 = 4KB)
  2. 캐시 히트: 사용자 버퍼에 복사 후 반환 (Zero syscall I/O 없음)
  3. 캐시 미스: 블록 레이어에 I/O 요청
                → 페이지 캐시에 로드 → 사용자 버퍼에 복사

Readahead (선독):
  순차 읽기 감지 → 다음 페이지 미리 로드
  /sys/block/sda/queue/read_ahead_kb (기본 128KB)
  대용량 순차 I/O: read_ahead_kb 증가로 성능 향상
  랜덤 I/O: readahead 불필요 → 낭비 (0으로 설정 가능)
```

### 쓰기 경로 (Writeback)

```
write() 시스템 콜:
  1. 페이지 캐시에 쓰기 (즉시 반환)
  2. 페이지를 Dirty로 표시
  3. 백그라운드 kworker/flusher가 주기적으로 디스크에 flush

Dirty 페이지 관리:
  vm.dirty_background_ratio: 전체 메모리의 X% Dirty → 백그라운드 flush 시작
  vm.dirty_ratio: 전체 메모리의 X% Dirty → write() 블로킹 시작
  vm.dirty_expire_centisecs: X*0.01초 지난 Dirty 페이지 → flush 대상
  vm.dirty_writeback_centisecs: kworker wake-up 주기

  기본값:
    dirty_background_ratio=10
    dirty_ratio=20
    dirty_expire_centisecs=3000 (30초)

  DB 서버 권장:
    dirty_background_ratio=3    # 빠른 flush
    dirty_ratio=10
    dirty_expire_centisecs=500
```

### Direct I/O vs Buffered I/O

```
Buffered I/O (기본):
  페이지 캐시 경유 → 반복 읽기 빠름
  쓰기 즉시 반환 → 처리량 높음
  단점: 커널-사용자 공간 복사 비용, 캐시 컨트롤 불가

Direct I/O (O_DIRECT):
  페이지 캐시 bypass → 직접 디바이스
  DB처럼 자체 버퍼 관리 시 이중 캐싱 방지
  정렬 요구: 오프셋과 버퍼가 512 또는 4096 배수
  사용: MySQL InnoDB, PostgreSQL (일부), Oracle

비교:
  순차 대용량 읽기: Direct I/O ≈ Buffered (readahead로 동등)
  반복 읽기: Buffered I/O 압도적 우위 (캐시)
  DB 버퍼풀: Direct I/O (이중 캐싱 방지)

mmap:
  파일을 가상 주소에 매핑 → 포인터로 직접 접근
  페이지 폴트 시 페이지 캐시에서 로드
  SQLite, LMDB가 사용
```

---

## 3. 블록 레이어와 I/O 스케줄러

### Multi-Queue Block Layer (blk-mq)

```
기존 단일 큐 → blk-mq (멀티 큐):
  CPU 코어마다 별도 제출 큐 (Submission Queue)
  → CPU 간 잠금 경쟁 제거
  → NVMe의 수십 개 하드웨어 큐 활용

큐 구조:
  Software Queue (코어당)
    → Hardware Dispatch Queue (디바이스당)
      → 디바이스 하드웨어 큐 (NVMe: 최대 65535)
```

### I/O 스케줄러

```
none (noop):
  큐잉 없이 순서대로 전달
  NVMe/SSD: 디바이스 자체가 최적화 → none 적합
  권장: NVMe SSD

mq-deadline:
  읽기/쓰기 데드라인 큐 분리
  기아 방지 (오래된 I/O 우선 처리)
  HDD, 혼합 워크로드에 적합

kyber:
  레이턴시 기반 (읽기 레이턴시 목표 유지)
  SSD, 고성능 저장소

bfq (Budget Fair Queue):
  프로세스별 공정 대역폭 보장
  데스크톱, 인터랙티브 시스템

확인 및 변경:
  cat /sys/block/nvme0n1/queue/scheduler
  echo none > /sys/block/nvme0n1/queue/scheduler
```

### I/O 병합 (Merging)

```
블록 레이어가 인접한 I/O 요청 병합:
  read(offset=100, size=4KB) + read(offset=104, size=4KB)
  → 하나의 8KB I/O로 병합

병합 유형:
  Front merge: 새 요청이 기존 요청 앞에 붙음
  Back merge: 새 요청이 기존 요청 뒤에 붙음

관련 설정:
  /sys/block/sda/queue/nr_requests: I/O 스케줄러 큐 깊이 (기본 64)
  /sys/block/sda/queue/max_sectors_kb: 최대 I/O 크기
```

---

## 4. NVMe 내부 동작

### NVMe 아키텍처

```
PCIe 버스 연결:
  CPU ←── PCIe 버스 ──→ NVMe 컨트롤러 ──→ NAND Flash

NVMe 큐 쌍 (Queue Pair):
  Submission Queue (SQ): 호스트 → 컨트롤러 명령 전달
  Completion Queue (CQ): 컨트롤러 → 호스트 완료 통보

  최대 65535개 큐 쌍 지원
  → CPU 코어당 큐 할당 가능 → 잠금 경쟁 없음

  기존 SATA (1 큐, 32 깊이) vs NVMe (65535 큐 × 65535 깊이)

Command 처리:
  1. 호스트가 SQ 테일 포인터 업데이트 (NVMe doorbell register에 쓰기)
  2. 컨트롤러가 SQ에서 명령 가져와 실행
  3. 완료 시 CQ에 완료 항목 추가 + MSI-X 인터럽트 (또는 폴링)
  4. 호스트가 CQ 처리 후 헤드 포인터 업데이트
```

### NVMe 폴링 모드 (io_poll)

```
인터럽트 기반 I/O:
  I/O 완료 → 하드웨어 인터럽트 → ISR → 프로세스 깨우기
  레이턴시: ~10µs

폴링 모드:
  I/O 완료 → CPU가 지속적으로 CQ 확인
  레이턴시: ~5µs (인터럽트 오버헤드 제거)
  단점: CPU를 지속 점유 (spinwait)

  초저레이턴시 NVMe (Optane 등)에 적합
  → 레이턴시 < 10µs이면 폴링이 유리

활성화:
  echo 1 > /sys/block/nvme0n1/queue/io_poll
  또는 O_DIRECT + io_uring poll 모드
```

### NVMe 성능 측정

```bash
# fio — 스토리지 벤치마크
# 순차 읽기
fio --name=seqread --rw=read --bs=128k --size=4G \
    --numjobs=4 --iodepth=32 --direct=1 --filename=/dev/nvme0n1

# 랜덤 4K 읽기 (IOPS 측정)
fio --name=randread --rw=randread --bs=4k --size=4G \
    --numjobs=8 --iodepth=64 --direct=1 --filename=/dev/nvme0n1

# 혼합 (70% 읽기, 30% 쓰기)
fio --name=mixed --rw=randrw --rwmixread=70 --bs=4k --size=4G \
    --numjobs=4 --iodepth=32 --direct=1

# NVMe 상태 확인
nvme smart-log /dev/nvme0n1
# critical_warning: 문제 있으면 비트 설정
# temperature: 온도
# available_spare: 예비 블록 남은 비율 (10% 이하 주의)
# percentage_used: 내구성 소진율 (100% 근접 시 교체)

# NVMe 큐 정보
cat /sys/block/nvme0n1/queue/nr_hw_queues   # 하드웨어 큐 수
cat /sys/block/nvme0n1/queue/nr_requests    # 큐 깊이
```

---

## 5. io_uring (Linux 5.1+)

### 기존 I/O 모델의 한계

```
read()/write() (동기):
  syscall → 블로킹 → 완료 대기 → 반환
  I/O당 2번 syscall (read + select/epoll)

AIO (비동기):
  io_submit → io_getevents 폴링
  O_DIRECT만 지원 (버퍼드 I/O 미지원)
  복잡한 인터페이스

epoll:
  소켓에 최적화, 파일 I/O에 부적합
  파일은 항상 "읽을 준비됨"으로 반환 → 실제 blocking
```

### io_uring 동작 원리

```
핵심 혁신: Submission/Completion Ring Buffer (커널-유저 공간 공유)

Submission Queue (SQE):
  사용자 공간이 SQE 배열에 I/O 요청 추가
  → io_uring_enter() syscall 한 번으로 다수 I/O 제출

Completion Queue (CQE):
  커널이 완료 시 CQE 배열에 추가
  → 사용자 공간이 폴링으로 확인 (syscall 없이!)

                  사용자 공간
    ┌──────────────────────────────────┐
    │  SQ Ring  │   CQ Ring  │  Buffers│
    └────────────────────────────────── ┘
         ↑ mmap           ↓ mmap
    ┌──────────────────────────────────┐
    │            커널 공간              │
    │  io_uring 인스턴스               │
    └────────────────────────────────── ┘

장점:
  배치: 여러 I/O를 한 번의 syscall로 제출
  폴링: CQ를 syscall 없이 폴링 (초저레이턴시)
  버퍼드 I/O 지원 (AIO와 달리)
  네트워크 소켓도 지원 (파일 + 소켓 통합)
  고정 파일/버퍼 등록으로 매핑 오버헤드 제거
```

### io_uring 사용 예시

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(32, &ring, 0);  // 32개 깊이

// 읽기 요청 제출
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, sizeof(buf), 0);
sqe->user_data = 1;  // 요청 식별자

io_uring_submit(&ring);  // 커널에 전달

// 완료 대기
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
printf("read %d bytes\n", cqe->res);
io_uring_cqe_seen(&ring, cqe);

io_uring_queue_exit(&ring);
```

```bash
# io_uring 사용 프로그램
# RocksDB: io_uring 지원 (--use-io-uring)
# liburing 기반 앱: nginx (패치), Node.js (libuv)

# io_uring 통계 확인
cat /proc/<pid>/fdinfo/<uring_fd>

# io_uring 이벤트 추적 (bpftrace)
bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe { @[args->opcode] = count(); }'
```

---

## 6. 파일시스템 내부

### 저널링 (Journaling)

```
문제: 파워 오프 시 메타데이터 불일치
  예: 파일 쓰기 중 → inode 업데이트 전 전원 끊김 → 고아 블록

저널링 모드:
  writeback: 메타데이터만 저널 (데이터는 페이지 캐시)
    → 가장 빠름, 파일 내용 손상 가능
  ordered (ext4 기본): 데이터 먼저 쓰기 후 메타데이터 저널
    → 균형 잡힌 안전성/성능
  journal: 데이터+메타데이터 모두 저널
    → 가장 안전, 2× 쓰기 비용

ext4 저널 옵션:
  mount -o data=ordered /dev/sda1 /mnt

저널 크기:
  /proc/fs/ext4/sda1/journal_info
  tune2fs -l /dev/sda1 | grep "Journal size"
  너무 작으면 저널 가득 차서 I/O 블로킹
```

### WAL vs 저널링

```
DB WAL:
  변경 사항을 WAL 파일에 먼저 순차 기록
  → 임의 I/O → 순차 I/O로 변환 (SSD에서도 유리)
  → 장애 복구: WAL 재생

파일시스템 저널링:
  메타데이터(또는 데이터) 변경을 저널에 기록
  → fsck 없이 빠른 복구

차이:
  WAL: DB 레벨, 트랜잭션 재생 가능
  저널링: FS 레벨, 파일시스템 일관성 복구
```

### Extent 기반 블록 할당 (ext4, XFS)

```
기존 블록 매핑 (ext2/ext3):
  12개 직접 포인터 + 간접 포인터 트리
  단편화 심함, 대용량 파일에서 비효율

Extent 기반 (ext4, XFS):
  extent = (논리 블록 시작, 물리 블록 시작, 길이)
  연속된 블록을 하나의 레코드로 표현
  예: 1GB 연속 파일 = extent 하나 (블록 262144개)

XFS B-Tree:
  extents가 많으면 B-Tree로 관리
  → 대용량 파일, 많은 단편화 시에도 효율적

단편화 확인:
  e2freefrag /dev/sda1       # ext4
  xfs_db -c frag /dev/sda1   # XFS
  filefrag -v /path/to/file  # 파일별 단편화
```

### XFS vs ext4 vs Btrfs

```
ext4:
  안정성: 최고 (가장 오래됨)
  대용량 파일: 좋음 (Extent 기반)
  최대 파일시스템: 1EB, 파일: 16TB
  권장: 일반 서버, 안정성 최우선

XFS:
  병렬 I/O: 최고 (병렬 Allocation Groups)
  대용량: 최고 (최대 8EB)
  작은 파일: ext4보다 약간 불리
  권장: 대용량 데이터, DB, 미디어 스토리지

Btrfs:
  CoW(Copy-on-Write): 스냅샷, RAID 내장
  체크섬: 데이터 무결성 자동 검증
  성능: ext4/XFS보다 일반적으로 낮음
  권장: 스냅샷 활용, 홈 서버
  주의: DB 워크로드에서 CoW 오버헤드

tmpfs:
  메모리 기반 파일시스템 (RAM 사용)
  /dev/shm, /tmp에 주로 사용
  재부팅 시 데이터 소실
```

---

## 7. AWS EBS 내부 동작

### EBS 볼륨 타입 비교

```
gp3 (General Purpose SSD, 3세대):
  기본: 3000 IOPS, 125 MB/s
  최대: 16000 IOPS, 1000 MB/s (독립 조정 가능)
  레이턴시: single-digit ms
  비용: gp2보다 20% 저렴
  권장: 대부분의 워크로드

gp2:
  IOPS: 볼륨 크기 × 3 (최소 100, 최대 16000)
  버스트: 3000 IOPS (크레딧 기반, < 1TB 볼륨)
  주의: 크레딧 고갈 시 기본 IOPS로 성능 하락

io2 Block Express:
  최대: 256000 IOPS, 4000 MB/s
  99.999% 내구성 (io1: 99.999%)
  sub-millisecond 레이턴시
  권장: Oracle RAC, SAP HANA, 고성능 DB

st1 (Throughput Optimized HDD):
  최대: 500 IOPS, 500 MB/s
  순차 I/O에 최적화
  권장: 빅데이터, 로그 처리

sc1 (Cold HDD):
  최대: 250 IOPS, 250 MB/s
  가장 저렴
  권장: 콜드 데이터 아카이브
```

### EBS 성능 최적화

```
EBS 최적화 인스턴스:
  일반 EC2: EBS와 네트워크가 대역폭 공유
  EBS 최적화: 전용 EBS 네트워크 경로 (추가 비용 없음, 최신 인스턴스 기본)

EBS 대역폭 vs 인스턴스 네트워크 대역폭:
  m5.xlarge: EBS 최대 4,750 Mbps (≈ 593 MB/s)
  볼륨이 아무리 빨라도 인스턴스 네트워크 대역폭이 병목 가능

IOPS 버스트 (gp2):
  초기 5.4M IOPS 크레딧 (약 30분 3000 IOPS)
  크레딧 충전: 볼륨 크기 × 3 IOPS/s 속도
  모니터링: BurstBalance 메트릭 (0이면 버스트 불가)
  → gp3로 마이그레이션 권장 (항상 일정 IOPS)
```

```bash
# EBS 성능 모니터링 (CloudWatch 메트릭)
# VolumeReadOps, VolumeWriteOps: IOPS
# VolumeReadBytes, VolumeWriteBytes: 처리량
# VolumeQueueLength: I/O 큐 깊이 (> 1이면 포화 시작)
# BurstBalance: gp2 크레딧 (%)

# 인스턴스에서 I/O 지연 확인
iostat -xz 1 /dev/nvme0n1
# await: EBS 왕복 레이턴시 포함 (네트워크 레이턴시 + 스토리지)

# 볼륨 타입 확인
aws ec2 describe-volumes --volume-ids vol-xxx \
  --query 'Volumes[].{Type:VolumeType,IOPS:Iops,Throughput:Throughput}'

# gp2 → gp3 변경 (다운타임 없음)
aws ec2 modify-volume --volume-id vol-xxx \
  --volume-type gp3 --iops 6000 --throughput 500
```

### EBS 스냅샷 내부

```
EBS 스냅샷:
  증분 스냅샷: 변경된 블록만 S3에 저장
  첫 스냅샷: 전체 볼륨
  이후: 이전 스냅샷 대비 변경 블록만

스냅샷 → 볼륨 복원:
  즉시 사용 가능 (Lazy Loading)
  첫 접근 시 S3에서 블록 로드 (레이턴시 높음)
  → 운영 전 사전 워밍 필요:
    dd if=/dev/nvme0n1 of=/dev/null bs=1M   # 전체 읽기로 워밍

EBS Fast Snapshot Restore:
  특정 AZ에서 스냅샷 사전 초기화
  → Lazy Loading 없이 즉시 최대 성능
  비용: 활성화 AZ × 시간당 요금

백업 전략:
  AWS Backup으로 스냅샷 자동화
  DLM(Data Lifecycle Manager): 보존 정책 자동 관리
  크로스 리전 복사: 재해 복구용
```

---

## 8. S3 내부 동작 원리

### S3 아키텍처 개요

```
S3 = 무한 확장 객체 스토리지

핵심 특성:
  결과적 일관성 → 2020년부터 강한 일관성 (read-after-write)
  11 nines 내구성 (99.999999999%)
  여러 AZ에 자동 복제 (Standard 스토리지 클래스)

내부 구조 (공개된 정보 기반):
  Lustre 기반 메타데이터 서비스 + 자체 분산 스토리지
  오브젝트 → 청크 분할 → 여러 노드에 분산 저장
  에라수어 코딩 (Erasure Coding)으로 내구성 확보
```

### S3 성능 최적화

```
요청 속도 제한 (버킷당):
  GET/HEAD: 5,500 requests/s/prefix
  PUT/POST/DELETE: 3,500 requests/s/prefix

Prefix 전략:
  단일 prefix: 병목
  날짜 기반 prefix: /2024/01/01/file → 날짜별 prefix
  랜덤 해시 prefix: /abc123/file → 높은 분산

멀티파트 업로드:
  5MB ~ 5GB 파트로 분할 병렬 업로드
  100MB 이상 파일: 멀티파트 필수 (단일 PUT 최대 5GB)
  최적 파트 크기: 100MB ~ 1GB (병렬도 × 파트 크기 = 처리량)

Transfer Acceleration:
  CloudFront 엣지 → AWS 내부 네트워크 → S3
  장거리 업로드 성능 향상 (30~300%)

S3 Express One Zone:
  단일 AZ, 10배 빠른 레이턴시 (한 자리 ms)
  ML 훈련 데이터, 고성능 분석에 적합
  내구성: 단일 AZ (Standard = 3 AZ)
```

### S3 스토리지 클래스

```
Standard:
  11 nines 내구성, 3+ AZ 복제
  즉시 접근, 비용 최고

Intelligent-Tiering:
  접근 패턴 자동 분석 → 적합 티어로 이동
  자동화 비용 있음 (객체당 월 0.0025$)

Standard-IA (Infrequent Access):
  접근 빈도 낮은 데이터
  스토리지 비용 低, 접근 비용 高
  최소 30일 보관, 128KB 최소 과금

Glacier Instant Retrieval:
  밀리초 검색 (IA와 동일)
  IA보다 스토리지 비용 낮음
  분기 1회 접근 기준

Glacier Flexible Retrieval:
  검색 시간: 분 ~ 시간
  Expedited(1~5분), Standard(3~5시간), Bulk(5~12시간)

Glacier Deep Archive:
  최저 비용, 검색 12~48시간
  7~10년 장기 보관

비용 최적화:
  S3 Storage Lens로 접근 패턴 분석
  Lifecycle Policy로 자동 티어 이동
  S3 Intelligent-Tiering for unknown access patterns
```

---

## 9. 스토리지 RAID와 Erasure Coding

### RAID 레벨 비교

```
RAID 0 (스트라이핑):
  분산: 데이터를 n개 디스크에 분산
  성능: 읽기/쓰기 n배
  내구성: 1개 디스크 고장 시 전체 손실
  용도: 임시 데이터, 극한 성능 필요 시

RAID 1 (미러링):
  복제: 동일 데이터를 2개 디스크에 저장
  성능: 읽기 2배, 쓰기 1배
  내구성: 1개 고장 시 정상
  용도: OS 디스크, 중요 시스템 파일

RAID 5:
  3개 이상 디스크, XOR 패리티
  내구성: 1개 고장 허용
  쓰기 패널티: 패리티 계산 필요 (Read-Modify-Write)
  SSD에서 쓰기 증폭 문제

RAID 6:
  2개 고장 허용 (이중 패리티)
  디스크 많을수록 유리

RAID 10 (1+0):
  미러링 + 스트라이핑
  성능 + 내구성 (각 미러에서 1개씩 고장 허용)
  비용 가장 높음 (50% 용량 낭비)
  권장: 고성능 DB, 높은 내구성 필요
```

### Erasure Coding

```
RAID보다 효율적인 내구성:
  Reed-Solomon 코드 사용
  n+k 방식: n개 데이터 청크 + k개 패리티 청크
  k개 청크까지 손실 허용

예시 (8+4):
  8개 데이터 + 4개 패리티 = 12개 청크
  4개 청크 손실까지 복구 가능
  저장 오버헤드: 12/8 = 1.5× (RAID 6 = 2×보다 효율적)

사용:
  S3, HDFS, Ceph, MinIO
  대용량 스토리지에서 RAID보다 비용 효율적

단점:
  읽기 시 디코딩 CPU 비용
  레이턴시 민감 워크로드에 불리
```

---

## 10. 실무 I/O 성능 진단

### iostat 해석

```bash
# 1초 간격 I/O 통계
iostat -xz 1

# 주요 필드:
# r/s, w/s: 읽기/쓰기 IOPS
# rkB/s, wkB/s: 읽기/쓰기 처리량 (KB/s)
# await: 평균 I/O 완료 시간 (ms)
# r_await, w_await: 읽기/쓰기 별 대기 시간
# %util: 디바이스 활용률 (100% = 포화)

# SSD에서의 해석:
# %util 100% = I/O 큐가 항상 찬 상태 (포화 징후)
# await 급증 = 스로틀링 또는 디바이스 성능 한계
# HDD: %util 100% 이면 확실한 병목
# SSD/NVMe: %util 100%여도 큐 깊이가 낮으면 실제 여유 있을 수 있음
```

### 실전 진단 시나리오

```bash
# 시나리오: 앱 응답이 느려짐, DB 서버 I/O 의심

# 1단계: 시스템 전반 확인
iostat -xz 1
# → /dev/nvme0n1 await > 10ms, %util > 80% 확인

# 2단계: 어느 프로세스가 I/O 유발하는지
iotop -o -b -n3
# 또는
pidstat -d 1 | head -20

# 3단계: 파일 레벨로 좁히기
lsof +D /path/to/datadir | head -20
# DB 데이터 파일 중 무엇이 집중적으로 접근되는지

# 4단계: 블록 레이어 레이턴시
biolatency-bpfcc -d nvme0n1
# 히스토그램으로 레이턴시 분포 확인

# 5단계: 느린 파일 작업
fileslower-bpfcc 10
# 10ms 이상 파일 작업 실시간 출력

# 6단계: 페이지 캐시 히트율 확인
cachestat-bpfcc 1
# HITS: 페이지 캐시 히트
# MISSES: 디스크 I/O 필요
# DIRTIES: Dirty 페이지 생성
# RATIO: 캐시 히트율
```

### I/O 성능 튜닝 체크리스트

```bash
# NVMe SSD 최적화
echo none > /sys/block/nvme0n1/queue/scheduler
echo 32 > /sys/block/nvme0n1/queue/nr_requests   # 큐 깊이 조정

# 읽기 선독 비활성화 (랜덤 I/O 워크로드)
echo 0 > /sys/block/nvme0n1/queue/read_ahead_kb

# Writeback 튜닝 (DB 서버)
sysctl -w vm.dirty_background_ratio=3
sysctl -w vm.dirty_ratio=10
sysctl -w vm.dirty_expire_centisecs=500

# ext4 마운트 옵션 최적화
# noatime: 파일 접근 시간 기록 비활성화 (읽기 I/O 감소)
# data=ordered: 기본 저널 모드
# barrier=0: 전원 보호 장치 있으면 (UPS, RAID BBU)
mount -o remount,noatime,data=ordered /data

# /etc/fstab
# /dev/nvme0n1p1 /data ext4 defaults,noatime,data=ordered 0 2
```

---

## 11. 실무 심화 포인트 & 면접 대비

### "Direct I/O vs Buffered I/O 어떻게 선택하는가?"
```
Buffered I/O:
  반복 읽기 많음 (OS 페이지 캐시 활용)
  쓰기 처리량 우선 (Writeback)
  일반 파일, 웹 서버 정적 파일

Direct I/O:
  자체 캐시 관리 (InnoDB Buffer Pool, Redis)
  이중 캐싱 방지
  정렬된 버퍼 필요 (4KB 배수)
  대용량 순차 스트리밍 (페이지 캐시 오염 방지)
```

### "io_uring이 epoll보다 좋은 경우는?"
```
epoll이 유리:
  네트워크 소켓 다수 (N개 FD 모니터링)
  I/O 준비 이벤트 방식에 최적

io_uring이 유리:
  파일 I/O (epoll은 파일 FD 미지원)
  고IOPS 워크로드 (syscall 오버헤드 제거)
  배치 처리 (여러 I/O를 한 syscall로)
  버퍼드 + Direct I/O 혼용

주목할 사용례:
  io_uring + 고정 버퍼 = 제로카피 I/O
  io_uring + 폴링 = NVMe Optane 수준 저레이턴시
```

### "EBS gp2 vs gp3 마이그레이션 판단 기준"
```
gp3로 마이그레이션 권장:
  BurstBalance 메트릭이 자주 낮아지는 경우
  볼륨 크기 < 334GB (gp2 기본 IOPS < 1000)
  IOPS를 독립적으로 조정하고 싶은 경우
  비용 절감 (gp3가 gp2 대비 20% 저렴)

마이그레이션:
  다운타임 없음 (온라인 변경)
  수분 ~ 수 시간 소요
  변경 중에도 성능 유지
```

### "S3 eventually consistent → strong consistent 변경 의미"
```
기존 (eventual):
  PUT 후 즉시 GET → 이전 버전 반환 가능
  → 캐싱 레이어 필요, 복잡한 로직

현재 (strong, 2020년~):
  PUT 후 즉시 GET → 새 버전 보장
  → 더 단순한 애플리케이션 로직
  → 추가 비용 없음

단, 버킷 목록 일관성:
  ListObjects는 여전히 최종적 일관성 가능성
  (분산 메타데이터 특성상)
```

| 질문 | 핵심 답변 |
|------|-----------|
| 블록 I/O vs 파일 I/O | 블록=raw 디바이스, 파일=파일시스템 추상화 경유 |
| 저널링 모드 3가지 | writeback(메타만), ordered(데이터후메타), journal(모두) |
| NVMe가 SATA보다 빠른 이유 | PCIe 직결, 멀티큐(65535), 낮은 프로토콜 오버헤드 |
| io_uring 핵심 혁신 | 공유 링 버퍼로 syscall 최소화, 파일+소켓 통합 |
| EBS 볼륨 타입 선택 | gp3=기본, io2=고성능DB, st1=순차대용량, sc1=아카이브 |
| S3 11 nines 내구성 구현 | 멀티AZ + Erasure Coding으로 다수 노드 장애 허용 |
