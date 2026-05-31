# JVM 메모리 구조

## JVM 메모리 영역 전체 그림

```
┌───────────────────────────────────────────────────────┐
│                    JVM Process Memory                  │
│                                                       │
│  ┌──────────────────────────────────────────────┐    │
│  │                  Heap                         │    │
│  │  ┌────────────────┐  ┌────────────────────┐  │    │
│  │  │  Young Gen     │  │    Old Gen         │  │    │
│  │  │  ┌───┬───┬───┐ │  │  (Tenured)         │  │    │
│  │  │  │ E │ S0│ S1│ │  │                    │  │    │
│  │  │  └───┴───┴───┘ │  │                    │  │    │
│  │  └────────────────┘  └────────────────────┘  │    │
│  └──────────────────────────────────────────────┘    │
│                                                       │
│  ┌──────────┐  ┌──────────────────────────────┐     │
│  │ Metaspace│  │   Thread Stack (스레드별)      │     │
│  │ (클래스  │  │  ┌────┐  ┌────┐  ┌────┐      │     │
│  │  메타)   │  │  │ S1 │  │ S2 │  │ S3 │ ...  │     │
│  └──────────┘  │  └────┘  └────┘  └────┘      │     │
│                └──────────────────────────────┘     │
│  ┌──────────────────────────────────────────────┐    │
│  │    Native Memory (Direct Buffer, JNI, etc)   │    │
│  └──────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────┘
```

---

## Heap (힙)

모든 **객체와 배열**이 저장되는 공간. GC의 주 관리 대상.

### Young Generation (젊은 세대)

```
Eden Space  : 새 객체 생성 위치 (대부분의 객체가 여기서 생애 마감)
Survivor 0  : Minor GC 생존 객체 이동
Survivor 1  : S0/S1 번갈아 사용 (항상 하나는 비어 있음)
```

**Minor GC 흐름**:
```
① Eden 가득 참 → Minor GC 발동
② Eden + Survivor(활성) 살아있는 객체 → 다른 Survivor로 복사
③ 각 객체의 age(생존 횟수) 증가
④ age >= threshold (기본 15) → Old Gen으로 Promotion
⑤ Eden, 이전 Survivor 전체 비움
```

- **빠른 GC**: Young Gen은 작고 대부분 죽어있음 → Minor GC는 수 ms
- **Stop-the-World(STW)**: Minor GC 중에도 애플리케이션 스레드 중단

### Old Generation (노년 세대)

- 오래 살아남은 객체 (age 초과) 또는 큰 객체 저장
- **Major GC / Full GC**: Old Gen GC → 수십 ms ~ 수 초 STW
- Full GC 잦음 = 힙 구조 또는 메모리 누수 문제

### 힙 설정

```bash
-Xms2g          # 초기 힙 크기
-Xmx4g          # 최대 힙 크기
-Xmn512m        # Young Gen 크기 (또는 -XX:NewSize/-XX:MaxNewSize)
-XX:SurvivorRatio=8  # Eden:Survivor = 8:1:1 (기본값)

# -Xms = -Xmx 권장 (힙 크기 변동 없앰 → GC 오버헤드 감소, 예측 가능한 성능)
```

---

## Stack (스레드 스택)

각 스레드마다 **독립적인 스택** 보유.

```
Thread Stack (스레드별)
┌─────────────────────────┐
│   Stack Frame N (최신)  │  ← 현재 실행 중인 메서드
│   - 지역 변수           │
│   - 매개변수            │
│   - 리턴 주소           │
│   - 피연산자 스택        │
├─────────────────────────┤
│   Stack Frame N-1       │
├─────────────────────────┤
│   Stack Frame 1         │  ← main() 등 최초 호출
└─────────────────────────┘
```

- **기본 크기**: 256KB ~ 1MB (OS/JVM마다 다름)
- `-Xss512k`: 스택 크기 조정 (스레드 수 많을 때 줄여서 메모리 절약)
- **StackOverflowError**: 재귀 호출 깊이 초과

### 스레드 수 × 스택 크기 = 스레드 스택 총 메모리
```
1000 threads × 1MB/thread = 1GB 스택 메모리
→ Tomcat/스레드 풀 크기 설정 시 고려 필요
```

---

## Metaspace (메타스페이스)

클래스 메타데이터 (클래스 구조, 메서드 정보, 상수 풀 등) 저장.

- **Java 7 이전**: PermGen (힙 내부, 고정 크기) → `OutOfMemoryError: PermGen space`
- **Java 8+**: Metaspace (Native Memory, 동적 크기) → 기본적으로 OS 메모리까지 확장 가능

```bash
-XX:MetaspaceSize=256m      # 초기 크기 (이 크기 도달 시 GC 발동)
-XX:MaxMetaspaceSize=512m   # 최대 크기 (미설정 시 무제한 → OOM 위험)
```

### Metaspace OOM 원인
- 클래스가 지속적으로 로딩 (동적 프록시, Reflection, JSP, 재배포)
- ClassLoader Leak: 앱 재배포 시 이전 ClassLoader가 참조 유지 → GC 불가
- `jstat -gcmetacapacity <PID>`로 증가 추세 모니터링

---

## Native Memory

JVM 힙 외부, OS에서 직접 관리하는 메모리.

| 영역 | 설명 |
|------|------|
| Direct Buffer | `ByteBuffer.allocateDirect()`, 파일 I/O, 네트워크 버퍼 |
| JNI | 네이티브 라이브러리 메모리 |
| Code Cache | JIT 컴파일된 네이티브 코드 |
| Thread Stack | 각 스레드의 스택 |
| GC 메타데이터 | GC 알고리즘 내부 자료구조 |

### Native Memory 모니터링 (NMT)
```bash
# NMT 활성화 (JVM 시작 시)
-XX:NativeMemoryTracking=summary   # 요약
-XX:NativeMemoryTracking=detail    # 상세

# 현재 Native Memory 사용량 조회
jcmd <PID> VM.native_memory summary
jcmd <PID> VM.native_memory detail
```

### Direct Buffer 주의사항
```java
// 힙 바깥에 할당됨 → GC 대상 아님
ByteBuffer buf = ByteBuffer.allocateDirect(1024 * 1024);

// 해제는 명시적으로 (또는 참조 해제 후 GC 시 정리)
// 누수 시 → java.lang.OutOfMemoryError: Direct buffer memory
```

---

## PC Register & Native Method Stack

- **PC (Program Counter) Register**: 현재 실행 중인 바이트코드 주소 (스레드별)
- **Native Method Stack**: JNI 네이티브 메서드 실행 스택 (C 스택)

---

## 메모리 문제 진단

### OOM (OutOfMemoryError) 유형별 원인

| OOM 메시지 | 원인 |
|-----------|------|
| `Java heap space` | 힙 부족 또는 메모리 누수 |
| `GC overhead limit exceeded` | GC에 98% 이상 시간 소비 (힙 부족) |
| `Metaspace` | 클래스 로딩 과다, ClassLoader 누수 |
| `Direct buffer memory` | DirectByteBuffer 누수 |
| `unable to create new native thread` | 스레드 수 한도 초과 (ulimit) |
| `PermGen space` | Java 7 이하, PermGen 부족 |

### Heap Dump 분석

```bash
# OOM 시 자동 Heap Dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# 수동 생성
jmap -dump:format=b,file=heap.hprof <PID>

# 빠른 객체 통계 (Heap Dump 없이)
jmap -histo:live <PID> | head -30

# 분석 도구
# Eclipse MAT (Memory Analyzer Tool) → .hprof 열기
# VisualVM → 실시간 힙 모니터링
# YourKit, JProfiler → 상용
```

### 메모리 누수 탐지 패턴

```bash
# 힙 사용량 추세 모니터링 (1초 간격)
jstat -gcutil <PID> 1000
# Old Gen(O 컬럼)이 GC 후에도 계속 증가 → 누수 의심

# RSS 증가 추세 확인 (힙 + 네이티브)
watch -n 5 "ps -o pid,rss,vsz --pid <PID>"

# GC 로그에서 Old Gen 기저선 확인
# Full GC 후에도 Used Old Gen이 계속 증가 → 누수
```

---

## 실무 포인트

- **RSS > Xmx**: JVM 힙 외 Native Memory(Direct Buffer, JNI, 스레드 스택)도 포함됨. 컨테이너 메모리 제한은 RSS 기준 → `Xmx`를 컨테이너 제한의 75~80%로 설정
- **`-Xms = -Xmx`**: 힙 크기 동적 변경 없앰 → GC 오버헤드 감소, 예측 가능한 성능. 프로덕션 권장
- **Young Gen 비율**: 단기 객체 많은 서비스 → Young Gen 크게 (Minor GC 빈도 줄임). `NewRatio=2` 기본 (Old:Young = 2:1)
- **Survivor 크기**: `-XX:SurvivorRatio=8` 기본 (Eden:S0:S1 = 8:1:1). Survivor 너무 작으면 조기 Old Gen Promotion → Major GC 증가
- **컨테이너 힙 자동 설정**: Java 10+ → `-XX:+UseContainerSupport` (기본 활성화). 컨테이너 메모리 제한 인식. `-XX:MaxRAMPercentage=75.0`으로 비율 설정
- **Direct Memory 한도**: `-XX:MaxDirectMemorySize=1g`. Netty, Kafka Producer/Consumer가 많이 사용
