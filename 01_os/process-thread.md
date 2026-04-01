# 프로세스와 스레드

## 프로세스 vs 스레드

| 구분 | 프로세스 | 스레드 |
|------|---------|--------|
| 메모리 | 독립된 주소 공간 | 프로세스 내 메모리 공유 |
| 생성 비용 | 높음 (fork) | 낮음 |
| 통신 | IPC (파이프, 소켓, 공유메모리) | 공유 메모리 직접 접근 |
| 격리 | 강함 (하나 죽어도 다른 것 영향 없음) | 약함 (하나 크래시 → 전체 위험) |
| Context Switch | 느림 | 빠름 |

---

## 프로세스 상태

```
Running → Waiting → Ready → Running (반복)
              ↓
           Zombie (부모가 wait() 안 한 경우)
```

- **Running**: CPU에서 실행 중
- **Waiting (Blocked)**: I/O 완료 대기
- **Ready**: CPU 할당 대기
- **Zombie**: 종료됐지만 부모가 exit status를 읽지 않음
- **Orphan**: 부모가 먼저 죽은 프로세스 (init/systemd가 입양)

---

## Context Switching

CPU가 다른 프로세스/스레드로 전환할 때:
1. 현재 프로세스 레지스터 상태(PCB)를 저장
2. 다음 프로세스 PCB를 복원
3. 실행 재개

**비용**: 캐시 무효화(Cache Miss)가 주요 오버헤드

---

## fork() vs exec()

```c
pid_t pid = fork();  // 현재 프로세스 복제

if (pid == 0) {
    // 자식 프로세스
    exec("/bin/ls", ...);  // 새 프로그램으로 교체
} else {
    // 부모 프로세스
    wait(NULL);  // 자식 종료 대기
}
```

- `fork()`: 현재 프로세스 전체 복제 (Copy-on-Write)
- `exec()`: 현재 프로세스를 새 프로그램으로 교체
- Shell이 명령어 실행할 때 fork → exec 패턴 사용

---

## IPC (Inter-Process Communication)

| 방식 | 특징 | 사용 사례 |
|------|------|---------|
| Pipe | 단방향, 부모-자식 간 | shell 파이프 (`\|`) |
| Named Pipe (FIFO) | 파일시스템 기반 | 관련 없는 프로세스 간 |
| Unix Socket | 양방향, 빠름 | nginx ↔ PHP-FPM |
| TCP Socket | 네트워크 가능 | 마이크로서비스 |
| Shared Memory | 가장 빠름 | Redis, memcached 내부 |
| Message Queue | 비동기, 버퍼링 | 작업 큐 |
| Signal | 비동기 이벤트 알림 | 종료/재로드 |

---

## 스레드 동기화 문제

### Race Condition
```
Thread A: x = x + 1  (read x=0, write x=1)
Thread B: x = x + 1  (read x=0, write x=1)  ← 결과: x=1 (예상: x=2)
```

### 해결 방법
- **Mutex**: 한 번에 하나의 스레드만 임계 구역 진입
- **Semaphore**: N개의 스레드 동시 접근 허용
- **Atomic Operation**: 분리 불가능한 단일 연산

### Deadlock 조건 (4가지 모두 충족 시 발생)
1. **Mutual Exclusion**: 자원을 하나의 프로세스만 사용
2. **Hold and Wait**: 자원을 가진 채로 다른 자원 대기
3. **No Preemption**: 자원을 강제로 빼앗을 수 없음
4. **Circular Wait**: A→B→C→A 순환 대기

---

## 실무 포인트

- **Nginx**: 멀티 프로세스 (Master + Worker). Worker 수 = CPU 코어 수
- **Node.js**: 단일 스레드 이벤트 루프 + libuv 스레드풀(I/O)
- **Go**: goroutine = 경량 스레드 (M:N 스케줄링)
- **컨테이너**: 프로세스 격리 (namespace + cgroup). 스레드보다 비용은 높지만 프로세스보다 가벼운 격리
- **CPU Affinity**: `taskset -c 0,1 ./app` — 특정 코어에 프로세스 고정하여 캐시 효율 향상
- **Go/Java의 GC 멈춤 (Stop-the-world)**: 짧은 레이턴시 요구 서비스에서 튜닝 필요
