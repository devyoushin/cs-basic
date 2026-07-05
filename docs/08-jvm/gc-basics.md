# GC 기초 (Garbage Collection)

## GC의 역할

Java는 메모리를 자동으로 관리. GC가 **더 이상 참조되지 않는 객체(가비지)**를 탐지하고 메모리 회수.

```
객체 생성 → 참조 유지 → ... → 참조 해제 → 가비지
                                              ↓
                                          GC가 탐지 & 회수
```

---

## GC 기본 알고리즘

### 1. Mark & Sweep

```
Phase 1: Mark (마킹)
  GC Root (스택, 스태틱 변수, JNI 참조 등)에서 시작
  → 참조 추적 (DFS/BFS)
  → 도달 가능한(Reachable) 객체에 마킹

Phase 2: Sweep (수거)
  → 마킹 안 된 객체 메모리 해제

Phase 3: Compact (압축, 선택적)
  → 살아있는 객체를 한쪽으로 모아 단편화 제거
```

단점: Sweep 후 메모리 단편화, STW 발생

### 2. Copying (복사)
```
From Space [A][B][C][ ][ ]   →   To Space [ ][ ][ ][ ][ ]
→ 살아있는 객체만 To Space에 복사
→ From Space 전체 비움
→ 단편화 없음, 빠른 할당 (포인터만 증가)
```
Young Gen의 Eden → Survivor 복사 방식이 이 원리.

### 3. Mark & Compact
Mark → Compact (살아있는 객체를 한쪽으로 이동) → 단편화 제거
Old Gen에서 주로 사용.

### GC Root (GC 기준점)
- JVM 스택의 지역 변수 / 매개변수
- 클래스의 static 변수
- JNI 참조
- 활성 스레드

---

## Stop-the-World (STW)

GC 실행 중 **애플리케이션 스레드 전체 중단**.
- 객체 참조가 GC 중에 변경되면 일관성 깨짐 → 안전을 위해 중단
- **GC 튜닝의 핵심 목표**: STW 시간 최소화

```
App Threads: ──────────────────┤ STW ├─────────────────
GC Thread:                     │ GC  │
                                ↑              ↑
                          GC 시작 (중단)    GC 완료 (재개)
```

---

## JVM GC 종류

### Serial GC
```bash
-XX:+UseSerialGC
```
- 단일 스레드 GC
- 클라이언트 애플리케이션, 작은 힙 (수백 MB)
- 프로덕션 서버에 부적합

### Parallel GC (Throughput GC)
```bash
-XX:+UseParallelGC          # Java 8 이하 기본
-XX:ParallelGCThreads=4     # GC 스레드 수
```
- 여러 스레드로 Young/Old Gen 병렬 GC
- **처리량(Throughput) 최대화** 목표
- STW 시간 길 수 있음
- 배치 처리, 과학 계산에 적합

### G1 GC (Garbage-First GC)
```bash
-XX:+UseG1GC    # Java 9+ 기본
-XX:MaxGCPauseMillis=200   # 목표 최대 STW 시간 (ms)
-XX:G1HeapRegionSize=4m    # Region 크기 (1~32MB)
```

**핵심 개념: Region 기반 힙**
```
G1 힙 (Region으로 분할)
┌──┬──┬──┬──┬──┬──┬──┬──┐
│E │E │S │O │O │O │E │H │  E=Eden, S=Survivor
├──┼──┼──┼──┼──┼──┼──┼──┤  O=Old, H=Humongous
│O │O │ │E │O │S │O │H │  (빈칸=사용가능)
└──┴──┴──┴──┴──┴──┴──┴──┘
```
- 힙을 동일 크기 Region으로 분할 → Young/Old 구분이 논리적
- **Garbage가 많은 Region 우선 수거** (Garbage-First)
- STW 시간 예측 가능 (목표 시간 기반 Region 선택)
- **Humongous Region**: Region 크기의 50% 초과 객체 직접 Old에 배치

**G1 GC 단계**:
```
① Young GC (Minor GC): Eden Region → Survivor/Old 이동 (STW)
② Concurrent Marking: 백그라운드에서 Old Region 도달성 분석
③ Mixed GC: Young + 일부 Old Region 수거 (STW, 짧게)
④ Full GC: 극히 드문 경우에만 (메모리 부족 시)
```

### ZGC (Z Garbage Collector)
```bash
-XX:+UseZGC    # Java 15+ 프로덕션 지원
```
- **최대 STW < 1ms** (힙 크기에 무관)
- Load Barriers + Colored Pointers로 동시 압축
- 대용량 힙(수십~수백 GB) 저지연 서비스에 적합
- 처리량은 G1 대비 약간 낮음

### Shenandoah GC
```bash
-XX:+UseShenandoahGC   # RedHat/Amazon 빌드
```
- ZGC와 유사한 목표: 매우 짧은 STW
- Java 12+

---

## GC 성능 지표

### GC 로그 분석 (Java 9+)

```bash
# GC 로그 옵션
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level:filecount=5,filesize=20m

# 실시간 확인
tail -f /var/log/gc.log
```

### jstat으로 GC 모니터링

```bash
jstat -gcutil <PID> 1000   # 1초 간격
```

```
S0    S1    E     O     M    YGC  YGCT   FGC  FGCT   GCT
0.00 85.33 72.46 35.12 95.7  134  4.321   2   0.891  5.212

S0/S1: Survivor 0/1 사용률 (%)
E    : Eden 사용률 (%)
O    : Old Gen 사용률 (%)        ← 지속 증가 시 누수 의심
M    : Metaspace 사용률 (%)
YGC  : Young GC 횟수
YGCT : Young GC 총 시간 (초)
FGC  : Full GC 횟수             ← 빈번하면 문제
FGCT : Full GC 총 시간 (초)
```

### 핵심 관찰 포인트
```
1. Full GC 빈도: 1시간에 1회 이하 정상. 수 분 이내 반복 = 심각
2. Old Gen 회수율: Full GC 후 Old Gen이 줄어야 정상
   줄지 않으면 → 메모리 누수
3. STW 시간: -XX:MaxGCPauseMillis 목표 대비 초과 여부
4. GC 오버헤드: GCT / 전체 시간 × 100. 5% 이상이면 GC 과부하
```

---

## GC 튜닝 방향

### Step 1: 문제 정의
```
처리량 저하? → GC 오버헤드 (GCT) 확인
레이턴시 스파이크? → STW 시간 (FGCT, YGCT) 확인
OOM? → 힙 사용량 & 누수 확인
```

### Step 2: GC 로그 수집 & 분석
```bash
# GCEasy.io 또는 GCViewer로 로그 파일 분석
# → Pause Time 분포, Allocation Rate, Promotion Rate
```

### Step 3: 힙 크기 조정
```bash
# Old Gen Full GC 후 사용량 × 3 = 권장 Old Gen 크기
# Young Gen: Allocation Rate × 목표 GC 간격
```

### Step 4: GC 알고리즘 선택

| 목표 | 권장 GC |
|------|---------|
| 처리량 최대화 (배치) | Parallel GC |
| 일반 서버 (균형) | G1 GC |
| 저지연 (p99 < 10ms) | ZGC or Shenandoah |
| 메모리 작음 (< 1GB) | Serial GC |

---

## GC 관련 실무 시나리오

### Full GC Storm (연속 Full GC)
```
증상: CPU 100%, 응답 없음, GC 로그에 Full GC 반복
원인:
  1. 힙 너무 작음 → -Xmx 증가
  2. 메모리 누수 → Heap Dump 분석
  3. Old Gen Promotion 과다 → Young Gen 크기 확인
  4. Metaspace 부족 → -XX:MaxMetaspaceSize 설정

조치: jmap -histo:live <PID> → 상위 객체 타입 확인
```

### Promotion Failure
```
Young GC 중 Old Gen에 공간 없음 → Full GC 트리거
원인: Old Gen 단편화 or 용량 부족
해결: Old Gen 비율 늘리기, G1GC Mixed GC 더 자주 수행
```

### 긴 STW 시간
```
G1GC에서 목표 Pause 초과:
- -XX:MaxGCPauseMillis 값 현실화
- Region 크기 조정 (-XX:G1HeapRegionSize)
- Humongous 객체 줄이기 (Region 크기 조정 또는 객체 분할)
```

---

## 실무 포인트

- **GC 로그는 항상 활성화**: 프로덕션에서도 오버헤드 미미. OOM/성능 문제 사후 분석에 필수
- **Full GC = 경보**: Full GC 발생 시 알람 설정. 빈번하면 반드시 원인 분석 (힙 사이즈, 누수)
- **STW 시간 SLA 설정**: p99 레이턴시 목표 있다면 `MaxGCPauseMillis` 설정하고 모니터링
- **G1GC Mixed GC 조정**: `-XX:G1MixedGCCountTarget` (기본 8), `-XX:G1OldCSetRegionThresholdPercent` → Old Gen 회수 속도 조정
- **컨테이너 환경**: `-XX:+UseContainerSupport` (기본 활성) + `-XX:MaxRAMPercentage=75` 설정. cgroup 메모리 제한 인식
- **GC 분석 도구**: GCEasy.io (GC 로그 업로드), VisualVM, JFR + JDK Mission Control
- **Allocation Rate**: 빠른 객체 생성 → Eden 빨리 가득 참 → Minor GC 빈번. `jstat -gc <PID> 1s`의 Eden 사용률 증가 속도로 측정
