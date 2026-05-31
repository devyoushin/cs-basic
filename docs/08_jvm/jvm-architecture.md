# JVM 아키텍처

## JVM 전체 구조

```
┌──────────────────────────────────────────────────────┐
│                      JVM                             │
│  ┌───────────────────────────────────────────────┐  │
│  │           Class Loader Subsystem               │  │
│  │  Loading → Linking → Initialization            │  │
│  └───────────────────────────────────────────────┘  │
│            ↓                                         │
│  ┌─────────────────────────────────────────────┐    │
│  │           Runtime Data Areas                 │    │
│  │  Method Area │ Heap │ Stack │ PC │ Native Stack │ │
│  └─────────────────────────────────────────────┘    │
│            ↓                                         │
│  ┌───────────────────────────────────────────────┐  │
│  │           Execution Engine                     │  │
│  │  Interpreter → JIT Compiler → GC               │  │
│  └───────────────────────────────────────────────┘  │
│            ↓                                         │
│  ┌───────────────────────────────────────────────┐  │
│  │     Native Method Interface (JNI)              │  │
│  └───────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

---

## 클래스 로더 (Class Loader)

Java는 런타임에 .class 파일을 동적 로딩.

### 로딩 순서 (위임 모델: Parent Delegation)
```
Bootstrap ClassLoader (JVM 내장, C++ 구현)
    ↑ 위임
Extension ClassLoader → $JAVA_HOME/lib/ext
    ↑ 위임
Application ClassLoader → Classpath
    ↑ 위임
Custom ClassLoader (필요 시)
```

### 위임 모델의 의미
- 자식 로더가 먼저 부모에게 위임 → 부모가 못 찾으면 자식이 로드
- **보안**: `java.lang.String` 같은 핵심 클래스를 사용자 코드로 교체 불가
- **클래스 격리**: 웹 애플리케이션 서버(Tomcat)는 앱별 ClassLoader로 격리

### 클래스 로딩 3단계
```
① Loading  → .class 파일 읽어 메모리(Method Area)에 적재
② Linking
   - Verify   → 바이트코드 유효성 검사
   - Prepare  → static 변수 기본값 초기화
   - Resolve  → 심볼릭 참조 → 직접 참조로 변환
③ Initialization → static 블록 실행, static 변수 초기화
```

### 클래스 로딩 문제 진단
```bash
# 클래스 로딩 추적 (JVM 옵션)
-verbose:class           # 모든 클래스 로딩 출력
-XX:+TraceClassLoading   # 로딩된 클래스 경로 포함

# 클래스 충돌 확인
java -verbose:class -jar app.jar 2>&1 | grep "java.lang.String"
```

---

## 실행 엔진 (Execution Engine)

### 인터프리터 (Interpreter)
- 바이트코드를 **한 줄씩 해석하여 실행**
- 빠른 시작, 느린 실행
- JVM 시작 시 모든 코드를 먼저 인터프리터로 실행

### JIT 컴파일러 (Just-In-Time Compiler)
```
바이트코드
    ↓
[인터프리터] → 실행 횟수 카운팅 (프로파일링)
    ↓ 임계치 초과 (HotSpot 감지)
[JIT 컴파일] → 네이티브 머신 코드로 컴파일
    ↓
[캐시에 저장] → 이후 실행은 네이티브 코드 직접 실행
```

### HotSpot JIT 계층 (Tiered Compilation)
```
Level 0: 인터프리터
Level 1: C1 컴파일러 (빠른 컴파일, 적은 최적화)
Level 2: C1 + 제한적 프로파일링
Level 3: C1 + 전체 프로파일링
Level 4: C2 컴파일러 (느린 컴파일, 최적 최적화)
```

JVM 옵션:
```bash
-XX:+TieredCompilation      # 기본 활성화 (Java 8+)
-XX:CompileThreshold=10000  # JIT 임계 호출 횟수 (기본: 10000)
-XX:+PrintCompilation       # JIT 컴파일 로그 출력
```

### JIT 최적화 기법
- **인라이닝(Inlining)**: 메서드 호출을 호출부에 삽입 → 함수 콜 오버헤드 제거
- **루프 최적화**: 루프 언롤링, SIMD 벡터화
- **이탈 분석(Escape Analysis)**: 객체가 메서드 밖으로 나가지 않으면 힙이 아닌 스택에 할당
- **Dead Code 제거**: 실행되지 않는 코드 제거

### Warm-up 현상
```
JVM 시작 → 인터프리터로 느리게 실행
           → 호출 횟수 누적
           → JIT 컴파일 발동
           → 네이티브 코드로 빠른 실행 (warm-up 완료)
```
- **트래픽 ramp-up**: 배포 후 즉시 100% 트래픽 인입 시 높은 레이턴시 발생
- **해결**: JVM 워밍업 스크립트, GraalVM AOT 컴파일

---

## 네이티브 메서드 인터페이스 (JNI)

Java 코드에서 C/C++ 네이티브 라이브러리를 호출:
```java
public class OS {
    public native int getPid();  // C 구현
    static { System.loadLibrary("os"); }
}
```

- **용도**: 하드웨어 접근, 레거시 라이브러리, 성능 크리티컬 코드
- **주의**: JNI 에러는 JVM 크래시로 이어질 수 있음
- **성능**: JNI 호출 자체에 오버헤드 존재 (빈번한 호출 피할 것)

---

## WAS와 JVM 구동 원리

### Tomcat 클래스 로딩 구조
```
JVM ClassLoader (Bootstrap, Extension, System)
    └── Catalina ClassLoader (Tomcat 코어)
            └── Shared ClassLoader (공유 라이브러리)
                    └── WebApp ClassLoader (각 앱별, 격리)
```
- 각 웹앱이 독립된 ClassLoader → 앱 간 클래스 충돌 방지
- **ClassLoader Leak**: 앱 재배포 시 이전 ClassLoader가 GC 안 되는 문제 → Metaspace OOM

### JVM 시작 옵션 주요 항목
```bash
# 힙 크기
-Xms2g -Xmx2g     # 최소/최대 힙 (동일하게 설정 권장 - GC 오버헤드 감소)

# GC 선택
-XX:+UseG1GC      # G1 GC (Java 9+ 기본)
-XX:+UseZGC       # ZGC (Java 15+, 저지연)

# GC 로깅 (Java 9+)
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=20m

# OOM 시 Heap Dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# JIT 관련
-XX:+TieredCompilation
-XX:ReservedCodeCacheSize=256m  # JIT 코드 캐시 (기본 240MB, 부족 시 JIT 비활성화)

# Metaspace (클래스 메타데이터)
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

---

## JVM 진단 도구

```bash
# 프로세스 목록 및 JVM 인수
jps -v

# JVM 통계 (GC, 힙 등)
jstat -gc <PID> 1s       # 1초마다 GC 통계
jstat -gcutil <PID> 1s   # GC 비율

# 런타임 JVM 플래그 확인
jinfo -flags <PID>
jinfo -flag MaxHeapSize <PID>

# Heap Dump 생성
jmap -dump:format=b,file=heap.hprof <PID>
jmap -histo <PID> | head -30  # 클래스별 객체 수 (빠른 확인)

# Thread Dump
jstack <PID>
kill -3 <PID>  # SIGQUIT → stdout에 Thread Dump 출력

# JVM 모니터링 GUI
jconsole <PID>
jvisualvm   # (Java 8 이하)

# 최신: JFR (Java Flight Recorder)
jcmd <PID> JFR.start duration=60s filename=/tmp/app.jfr
jfr print /tmp/app.jfr
```

---

## 실무 포인트

- **JIT Warm-up**: 배포 직후 레이턴시 급증은 정상. Canary 배포 + 트래픽 점진 인입으로 완화
- **`-XX:ReservedCodeCacheSize`**: JIT 코드 캐시 고갈 시 컴파일 중단 → 갑자기 성능 저하. `jstat -compiler <PID>`로 확인
- **ClassLoader Leak**: 운영 중 `jstat -gcmetacapacity <PID>`의 Metaspace 지속 증가 → 앱 재배포 문제 의심
- **`-server` 플래그**: C2 컴파일러 활성화 (64비트 JVM에서 기본값)
- **AOT (Ahead-of-Time)**: GraalVM native-image → JVM 없이 실행, 빠른 시작. Serverless/컨테이너 환경에 유리
- **JFR (Java Flight Recorder)**: 프로덕션에서 오버헤드 낮게 지속 기록 (< 1%). OOM/성능 문제 사후 분석에 필수
