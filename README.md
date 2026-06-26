# Linux 프로세스 트러블슈팅 - 장애 분석 리포트

---

# [Bug] Memory Leak으로 인한 MemoryGuard 정책에 의한 프로세스 강제 종료

## 1. Description (현상 설명)

`agent-leak-app-x86` 실행 후 약 30초가 경과하면 `SELF-TERMINATED` 메시지와 함께 프로세스가 예고 없이 종료된다. 애플리케이션 내부의 메모리 보호 정책(MemoryGuard)에 의한 강제 종료 현상이 반복적으로 발생한다.

- **발생 시점**: 앱 실행 후 약 30초
- **발생 조건**: `MEMORY_LIMIT=256` 설정 시
- **증상**: 터미널에 `SELF-TERMINATED` 출력 후 프로세스 종료

---

## 2. Evidence & Logs (증거 자료)

### monitor.sh 관제 로그 (Before: MEMORY_LIMIT=256MB)

```
[2026-06-22 09:08:03] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:25MB  DISK:950G FIREWALL:N/A
[2026-06-22 09:08:05] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:50MB  DISK:950G FIREWALL:N/A
[2026-06-22 09:08:10] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:75MB  DISK:950G FIREWALL:N/A
[2026-06-22 09:08:12] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:100MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:16] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:125MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:18] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:150MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:22] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:175MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:24] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:200MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:28] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:225MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:30] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:250MB DISK:950G FIREWALL:N/A
[2026-06-22 09:08:34] PROCESS:agent-leak-app-x86 STATUS:NOT_RUNNING
```

→ 메모리가 25MB씩 선형적으로 증가하다 256MB 도달 후 프로세스 종료

### 프로그램 실행 로그 발췌

```
2026-06-22 09:08:29,820 [INFO]     [MemoryWorker] Current Heap: 250MB
2026-06-22 09:08:32,879 [INFO]     [MemoryWorker] Current Heap: 275MB
2026-06-22 09:08:32,879 [CRITICAL] [MemoryGuard]  Memory limit exceeded (275MB >= 256MB) / (Recommend Over 256MB)
2026-06-22 09:08:32,879 [CRITICAL] [MemoryGuard]  Self-terminating process 4767 to prevent system instability.
>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<
```

---

## 3. Root Cause Analysis (원인 분석)

- **현상 분석**: 애플리케이션 내부 MemoryWorker가 힙(Heap) 메모리를 25MB씩 지속적으로 할당하면서 해제하지 않는 메모리 누수(Memory Leak) 결함이 존재한다.
- **시스템 동작**: 물리 메모리 사용량이 `MEMORY_LIMIT(256MB)`에 도달하자 MemoryGuard 정책이 시스템 전체 불안정을 방지하기 위해 해당 프로세스를 강제 종료시켰다.
- **OS 원리**: 프로세스가 힙 메모리를 할당(malloc)하고 해제(free)하지 않으면 사용 가능한 물리 메모리가 점차 고갈된다. 이는 결국 시스템 전체의 안정성을 위협하며, OS의 OOM Killer 또는 앱 자체 보호 정책이 해당 프로세스를 강제 종료하게 된다.

---

## 4. Workaround & Verification (조치 및 검증)

### 조치 내용

`~/.bash_profile`의 `MEMORY_LIMIT` 값을 256MB → 512MB로 상향 조정하여 임시적으로 가용 메모리를 확보했다.

```bash
# Before - 초기 설정
export MEMORY_LIMIT=256

# After - 조치
sed -i 's/MEMORY_LIMIT=256/MEMORY_LIMIT=512/' ~/.bash_profile
source ~/.bash_profile
echo $MEMORY_LIMIT  # 512 확인
```

### Before & After 비교

| 항목 | Before (MEMORY_LIMIT=256MB) | After (MEMORY_LIMIT=512MB) |
|------|---------------------------|--------------------------|
| 종료 시점 | 약 30초 후 275MB에서 SELF-TERMINATED | 525MB 도달 시 Cache Flush 후 생존 |
| 프로세스 상태 | 종료 | 계속 실행 중 |
| 로그 | `SELF-TERMINATED (Memory Limit Exceeded)` | `MEMORY RECOVERED (Cache Cleared)` |

### After 관제 로그 발췌 (MEMORY_LIMIT=512MB)

```
[2026-06-22 09:12:14] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:525MB DISK:950G FIREWALL:N/A
[2026-06-22 09:12:20] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:25MB  DISK:950G FIREWALL:N/A  ← Cache Flush 후 복구
[2026-06-22 09:12:22] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:50MB  DISK:950G FIREWALL:N/A
```

### 근본 해결 제안

임시 조치(MEMORY_LIMIT 상향)는 장애 발생 시점을 늦출 뿐 근본 원인을 해결하지 못한다. 근본적인 해결을 위해서는 MemoryWorker 내부에서 주기적으로 불필요한 데이터를 해제하는 리팩토링이 필요하다.

---
---

# [Bug] CPU 과점유에 의한 Watchdog 보호 조치로 프로세스 강제 종료

## 1. Description (현상 설명)

`agent-leak-app-x86` 실행 후 일정 시간이 경과하면, CPU 사용률이 급격히 상승하고 `WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)` 메시지와 함께 프로세스가 종료된다. 이는 오류가 아닌 CPU 과점유 방지 정책(Watchdog)에 의한 시스템 보호 조치이다.

- **발생 시점**: 앱 실행 후 약 40초
- **발생 조건**: `CPU_MAX_OCCUPY=90` 설정 시 실제 CPU가 임계치 초과
- **증상**: `WATCHDOG: INITIATING EMERGENCY ABORT` 출력 후 프로세스 종료

---

## 2. Evidence & Logs (증거 자료)

### 프로그램 실행 로그 발췌 (Before: CPU_MAX_OCCUPY=90%)

```
2026-06-22 09:25:50,832 [INFO]     [CpuWorker] Started. Maximum CPU Limit: 90%
2026-06-22 09:25:50,833 [INFO]     [CpuWorker] Current Load: 5.00%
2026-06-22 09:26:06,436 [INFO]     [CpuWorker] Current Load: 28.03%
2026-06-22 09:26:12,671 [INFO]     [CpuWorker] Current Load: 38.93%
2026-06-22 09:26:22,025 [INFO]     [CpuWorker] Current Load: 44.38%
2026-06-22 09:26:25,143 [INFO]     [CpuWorker] Current Load: 47.74%
2026-06-22 09:26:28,260 [INFO]     [CpuWorker] Current Load: 52.01%
2026-06-22 09:26:28,362 [CRITICAL] [CpuWorker] CPU Threshold Violated! (52.00%)
>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<
```

### monitor.sh 관제 로그 (Before: CPU_MAX_OCCUPY=90%)

```
[2026-06-22 09:25:51] PROCESS:agent-leak-app-x86 CPU:5.0%  MEM:25MB  DISK:950G FIREWALL:N/A
[2026-06-22 09:26:00] PROCESS:agent-leak-app-x86 CPU:18.2% MEM:50MB  DISK:950G FIREWALL:N/A
[2026-06-22 09:26:10] PROCESS:agent-leak-app-x86 CPU:36.0% MEM:75MB  DISK:950G FIREWALL:N/A
[2026-06-22 09:26:20] PROCESS:agent-leak-app-x86 CPU:44.4% MEM:100MB DISK:950G FIREWALL:N/A
[2026-06-22 09:26:28] PROCESS:agent-leak-app-x86 CPU:52.0% MEM:125MB DISK:950G FIREWALL:N/A
[2026-06-22 09:26:30] PROCESS:agent-leak-app-x86 STATUS:NOT_RUNNING
```

---

## 3. Root Cause Analysis (원인 분석)

- **현상 분석**: CpuWorker가 CPU 부하를 점진적으로 증가시키는 로직을 수행하던 중, 실제 CPU 사용률이 `CPU_MAX_OCCUPY(90%)`의 내부 임계치를 초과했다.
- **Watchdog 동작 원리**: 애플리케이션 내부의 Watchdog은 CPU 사용률을 지속적으로 모니터링하다가 설정된 임계치를 초과하면 SIGTERM 신호를 프로세스에 보내 강제 종료한다.
- **OS 원리**: CPU는 여러 프로세스가 시분할(Time Sharing) 방식으로 공유하는 자원이다. 하나의 프로세스가 CPU를 과점유하면 다른 프로세스들이 CPU를 할당받지 못해 응답 지연 또는 타임아웃이 발생한다.

---

## 4. Workaround & Verification (조치 및 검증)

### 조치 내용

`~/.bash_profile`의 `CPU_MAX_OCCUPY` 값을 90% → 50%로 하향 조정하여 CPU 부하가 임계치를 초과하지 않도록 제한했다.

```bash
# Before - 초기 설정
export CPU_MAX_OCCUPY=90

# After - 조치
sed -i 's/CPU_MAX_OCCUPY=90/CPU_MAX_OCCUPY=50/' ~/.bash_profile
source ~/.bash_profile
echo $CPU_MAX_OCCUPY  # 50 확인
```

### Before & After 비교

| 항목 | Before (CPU_MAX_OCCUPY=90%) | After (CPU_MAX_OCCUPY=50%) |
|------|---------------------------|--------------------------|
| 종료 여부 | 52% 도달 시 SIGTERM → 종료 | 50%에서 cooldown → 생존 |
| 로그 | `WATCHDOG: INITIATING EMERGENCY ABORT` | `Peak reached (50.00%). Starting cooldown...` |
| 프로세스 상태 | 종료 | 계속 실행 중 |

### After 로그 발췌 (CPU_MAX_OCCUPY=50%)

```
2026-06-22 09:31:49,264 [INFO] [CpuWorker] Peak reached (50.00%). Starting cooldown...
2026-06-22 09:31:50,269 [INFO] [CpuWorker] Current Load: 50.00%
2026-06-22 09:31:53,386 [INFO] [CpuWorker] Current Load: 48.12%
2026-06-22 09:31:56,503 [INFO] [CpuWorker] Current Load: 42.39%
```

---
---

# [Bug] 멀티스레드 환경에서 교착상태(Deadlock) 발생으로 프로세스 무응답

## 1. Description (현상 설명)

`agent-leak-app-x86`을 `MULTI_THREAD_ENABLE=true` 상태로 실행하면, 프로세스가 종료되지 않고 PID가 유지되나 CPU/메모리 변화가 없고 로그 출력도 완전히 멈춘 무응답 상태가 지속된다.

- **발생 시점**: 앱 실행 후 약 8초
- **발생 조건**: `MULTI_THREAD_ENABLE=true` 설정 시
- **증상**: 프로세스는 살아있으나 CPU 0%, 메모리 고정, 로그 멈춤

---

## 2. Evidence & Logs (증거 자료)

### PID 존재 확인

```bash
$ ps -ef | grep agent-leak-app-x86 | grep -v grep
agent  13781  24  0 09:37 pts/1  00:00:00 /home/agent-leak-app-x86
agent  13782 13781  0 09:37 pts/1  00:00:00 /home/agent-leak-app-x86
```

### 스레드 상태 확인

```bash
$ ps -L -p 13781
  PID   LWP TTY          TIME CMD
13781 13781 pts/1    00:00:00 agent-leak-app-
```

→ TIME이 00:00:00으로 CPU를 전혀 사용하지 않음

### monitor.sh 관제 로그 (CPU/MEM 변화 없음)

```
[2026-06-22 09:38:47] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:525MB DISK:950G FIREWALL:N/A
[2026-06-22 09:38:49] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:525MB DISK:950G FIREWALL:N/A
[2026-06-22 09:38:51] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:525MB DISK:950G FIREWALL:N/A
[2026-06-22 09:38:53] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:525MB DISK:950G FIREWALL:N/A
[2026-06-22 09:38:55] PROCESS:agent-leak-app-x86 CPU:0.0% MEM:525MB DISK:950G FIREWALL:N/A
```

→ CPU 0%, MEM 525MB로 고정 → 아무것도 처리하지 않는 상태

### 마지막 로그 (Deadlock 발생 직전)

```
2026-06-22 09:37:26,007 [INFO] [Worker-Thread-1] LOCK ACQUIRED: [Shared_Memory_A]. (Holding...)
2026-06-22 09:37:26,008 [INFO] [Worker-Thread-2] LOCK ACQUIRED: [Socket_Pool_B]. (Holding...)
2026-06-22 09:37:28,020 [INFO] [Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)
2026-06-22 09:37:28,020 [INFO] [Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
```

→ 이후 로그 완전히 멈춤

---

## 3. Root Cause Analysis (원인 분석)

마지막 로그를 분석하면 두 스레드가 서로의 자원을 무한히 기다리는 순환 대기 상태임을 확인할 수 있다.

```
Worker-Thread-1: Shared_Memory_A 점유 → Socket_Pool_B 대기
Worker-Thread-2: Socket_Pool_B 점유  → Shared_Memory_A 대기
→ 서로가 서로를 기다리는 순환 대기 (Circular Wait)
```

| 조건 | 설명 | 이 케이스 |
|------|------|----------|
| 상호 배제 (Mutual Exclusion) | 한 번에 하나의 스레드만 자원 사용 | Lock 획득 시 다른 스레드 접근 불가 |
| 점유 대기 (Hold and Wait) | 자원을 가진 채로 다른 자원 대기 | Thread-1은 A를 가진 채 B를 대기 |
| 비선점 (No Preemption) | 다른 스레드 자원을 강제로 빼앗을 수 없음 | Lock은 자발적 해제 전까지 유지 |
| 순환 대기 (Circular Wait) | A→B→A 서로가 서로를 기다림 | Thread-1→B, Thread-2→A |

---

## 4. Workaround & Verification (조치 및 검증)

### 조치 내용

`~/.bash_profile`의 `MULTI_THREAD_ENABLE` 값을 `true` → `false`로 변경하여 멀티스레드 모드를 비활성화했다.

```bash
# Before - 초기 설정
export MULTI_THREAD_ENABLE=true

# After - 조치
sed -i 's/MULTI_THREAD_ENABLE=true/MULTI_THREAD_ENABLE=false/' ~/.bash_profile
source ~/.bash_profile
echo $MULTI_THREAD_ENABLE  # false 확인
```

### Before & After 비교

| 항목 | Before (MULTI_THREAD_ENABLE=true) | After (MULTI_THREAD_ENABLE=false) |
|------|----------------------------------|----------------------------------|
| 프로세스 상태 | PID 유지, 무응답 | 정상 동작 |
| CPU | 0% 고정 | 정상 증감 |
| 로그 | Worker-Thread BLOCKED 후 멈춤 | 정상 출력 지속 |
| 종료 | 수동 kill 필요 | 정상 종료 |

### After 로그 발췌 (MULTI_THREAD_ENABLE=false)

```
2026-06-22 09:42:16,114 [INFO] [Scheduler] All tasks completed.
2026-06-22 09:42:16,180 [INFO] [MemoryWorker] Current Heap: 25MB
2026-06-22 09:42:16,181 [INFO] [CpuWorker] Started. Maximum CPU Limit: 50%
2026-06-22 09:42:19,227 [INFO] [MemoryWorker] Current Heap: 50MB
```

---
---

# Q&A - 심화 질문 답변

---

## Q1. monitor.sh에서 메모리 증가 패턴을 추적하기 위해 사용한 명령어와 데이터 추출 방법은?

monitor.sh에서 메모리 수치를 추출하기 위해 아래 명령어를 사용했다.

```bash
MEM=$(grep "Current Heap" /var/log/agent-app/agent_app.log 2>/dev/null | tail -1 | grep -oP '\d+(?=MB)')
```

**단계별 설명:**

1. `grep "Current Heap" /var/log/agent-app/agent_app.log` → 앱 로그에서 "Current Heap"이 포함된 줄만 필터링
2. `tail -1` → 가장 최근 기록된 마지막 줄만 추출
3. `grep -oP '\d+(?=MB)'` → MB 앞의 숫자만 추출 (`-o`: 매칭된 부분만 출력, `-P`: Perl 정규식 사용, `\d+(?=MB)`: MB 앞에 오는 숫자)

**`ps`나 `/proc`이 아닌 앱 로그를 사용한 이유:**

이 바이너리는 Python 기반으로 동작하며, Python은 OS에 메모리를 한 번에 크게 할당해두고 내부 메모리 풀에서 관리한다. 따라서 `ps`의 RSS(Resident Set Size)나 `/proc/$PID/status`의 VmRSS는 실제 애플리케이션 레벨의 힙 사용량을 정확히 반영하지 못한다. 앱이 직접 측정하여 출력하는 "Current Heap" 값이 실제 메모리 누수를 추적하는 데 가장 정확한 지표이다.

---

## Q2. 프로세스의 CPU 사용률을 확인하기 위해 선택한 도구와 적용한 옵션의 의미는?

monitor.sh에서 CPU 사용률을 추출하기 위해 `ps` 명령어를 사용했다.

```bash
CPU=$(ps -p $PID --no-headers -o %cpu 2>/dev/null | awk '{print $1}')
```

**각 옵션 설명:**

| 옵션 | 의미 |
|------|------|
| `ps` | Process Status. 실행 중인 프로세스 정보를 출력하는 명령어 |
| `-p $PID` | 특정 PID의 프로세스만 조회 (전체 프로세스 목록 불필요) |
| `--no-headers` | 컬럼 헤더 줄 없이 값만 출력. 스크립트에서 파싱 용이 |
| `-o %cpu` | 출력할 컬럼을 CPU 사용률만으로 제한 |
| `2>/dev/null` | 에러 메시지를 /dev/null로 버려 불필요한 출력 제거 |
| `awk '{print $1}'` | 공백으로 구분된 첫 번째 필드(CPU 값)만 추출 |

**`top` 대신 `ps`를 선택한 이유:**

`top`은 인터랙티브 TUI 도구로 스크립트에서 자동화하기 어렵다. `ps`는 단순히 현재 상태의 스냅샷을 반환하므로 `watch`와 조합하여 주기적으로 값을 추출하기에 적합하다.

---

## Q3. 프로세스가 "살아있지만 멈춰있는 상태"를 진단한 도구와 판단 흐름은?

Deadlock을 진단할 때 아래 순서로 도구를 사용했다.

**1단계: 프로세스 존재 여부 확인**

```bash
ps -ef | grep agent-leak-app-x86 | grep -v grep
```

→ PID 13781이 존재함을 확인. "프로세스가 살아있다"는 사실 입증.

**2단계: CPU/MEM 변화 없음 확인**

```bash
cat /var/log/agent-app/monitor.log | tail -10
```

→ CPU:0.0%, MEM:525MB가 계속 동일하게 반복됨 확인. "아무것도 처리하지 않는다"는 사실 입증.

**3단계: 마지막 로그 확인**

```bash
cat /var/log/agent-app/agent_app.log | tail -20
```

→ `WAITING for ... (Status: BLOCKED)` 이후 로그가 완전히 멈춤 확인. "언제부터 멈췄는지" 시점 특정.

**4단계: 스레드 CPU 사용 시간 확인**

```bash
ps -L -p 13781
```

→ TIME 컬럼이 00:00:00으로 CPU를 전혀 사용하지 않음 확인.

**판단 흐름 요약:**

```
PID 존재 → 살아있음
CPU/MEM 고정 → 아무것도 안 함
로그 멈춤 → 특정 시점 이후 동작 없음
마지막 로그: BLOCKED → 스레드가 자원 대기 중
→ 결론: Deadlock
```

---

## Q4. 메모리 누수 발생 시 MemoryGuard가 프로세스를 강제 종료하는 이유는?

메모리 누수가 계속되면 다음과 같은 연쇄 작용이 발생하기 때문이다.

**연쇄 작용:**

```
1. 프로세스의 힙 메모리 증가
2. 물리 RAM 고갈
3. OS가 스왑(Swap, 디스크)을 사용 시작
4. 디스크 I/O는 RAM보다 수백 배 느림 → 시스템 전체 응답 급격히 저하
5. 다른 프로세스들도 메모리 할당 불가 → 연쇄 장애
6. 최악의 경우 OS 자체가 불안정해져 전체 서비스 다운
```

**MemoryGuard가 먼저 개입하는 이유:**

OS의 OOM Killer는 메모리가 완전히 고갈된 후에야 개입하며, 어떤 프로세스를 죽일지 예측하기 어렵다. 반면 MemoryGuard는 **임계치에 도달하는 순간 해당 프로세스를 직접 종료**하므로 아래 이점이 있다.

- 종료 전 로그를 남겨 원인 추적이 가능하다
- 다른 프로세스에 영향이 전파되기 전에 조기 차단된다
- 시스템 전체가 불안정해지기 전에 제어된 종료가 이루어진다

---

## Q5. CPU 과점유 시 단일 프로세스를 종료하는 것이 시스템 보호에 왜 필요한가?

CPU는 여러 프로세스가 시분할(Time Sharing) 방식으로 나눠쓰는 공유 자원이다. 하나의 프로세스가 CPU를 독점하면 다음과 같은 문제가 발생한다.

**문제 상황:**

```
웹 서버 프로세스가 CPU를 90% 점유
→ 스케줄러가 다른 프로세스에 CPU를 줄 수 없음
→ DB 쿼리 처리 지연 → 응답 타임아웃
→ 로그 수집 지연 → 장애 추적 불가
→ 헬스체크 응답 지연 → 로드밸런서가 서버를 "다운"으로 판단
→ 결국 단 하나의 프로세스 때문에 전체 서비스 장애
```

**단일 프로세스 종료의 이점:**

특정 프로세스를 SIGTERM으로 종료하면 해당 서비스 하나만 중단되고 나머지 프로세스들은 즉시 정상화된다. "하나를 희생해서 전체를 살리는" 트레이드오프이다. 실무에서는 이 프로세스를 종료 후 자동으로 재시작(systemd restart, k8s pod restart)하도록 구성하는 것이 일반적이다.

---

## Q6. Deadlock 발생 원리를 상호 배제와 순환 대기 개념으로 설명하면?

**상호 배제(Mutual Exclusion)란:**

하나의 자원은 동시에 하나의 스레드만 사용할 수 있다는 원칙이다. 이 미션에서 `Shared_Memory_A`와 `Socket_Pool_B`는 각각 Lock으로 보호되어 있어, Thread-1이 `Shared_Memory_A`의 Lock을 획득하면 Thread-2는 그 Lock이 해제될 때까지 접근할 수 없다.

```
Thread-1: LOCK ACQUIRED: [Shared_Memory_A] → Thread-2는 접근 불가
Thread-2: LOCK ACQUIRED: [Socket_Pool_B]   → Thread-1은 접근 불가
```

**순환 대기(Circular Wait)란:**

각 스레드가 다음 스레드가 점유한 자원을 기다리는 순환 구조이다.

```
Thread-1 → Socket_Pool_B 필요 (Thread-2가 점유 중)
Thread-2 → Shared_Memory_A 필요 (Thread-1이 점유 중)

Thread-1 → Thread-2 → Thread-1 (순환)
```

이 순환이 만들어지는 순간, 어떤 스레드도 자신이 가진 Lock을 먼저 해제할 이유가 없기 때문에 무한 대기 상태가 된다. 두 조건이 동시에 충족되어야만 Deadlock이 발생한다.

---

## Q7. 로그에서 스레드 간 순환 의존 관계(A→B, B→A)를 어떻게 파악했는가?

아래 순서로 로그를 분석하여 순환 의존 관계를 추론했다.

**Step 1: 로그가 멈춘 시점 확인**

```
2026-06-22 09:37:28,020 [INFO] [Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)
2026-06-22 09:37:28,020 [INFO] [Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
← 이후 로그 없음
```

**Step 2: 각 스레드의 보유 자원과 대기 자원 정리**

```
Thread-1: 보유 → Shared_Memory_A / 대기 → Socket_Pool_B
Thread-2: 보유 → Socket_Pool_B   / 대기 → Shared_Memory_A
```

**Step 3: 의존 그래프 작성**

```
Thread-1 ──(대기)──→ Socket_Pool_B ──(보유)──→ Thread-2
Thread-2 ──(대기)──→ Shared_Memory_A ──(보유)──→ Thread-1

결론: Thread-1 → Thread-2 → Thread-1 (순환 확인)
```

Thread-1이 필요한 자원을 Thread-2가 갖고 있고, Thread-2가 필요한 자원을 Thread-1이 갖고 있어 서로 해제하지 않는 한 영원히 대기하는 순환 구조가 명확히 드러났다.

---

## Q8. 실제 운영 서버에서 메모리 누수를 장애 발생 전에 탐지하려면 monitor.sh를 어떻게 개선하겠는가?

현재 monitor.sh는 단순히 현재 메모리 값을 기록하는 수준이다. 아래와 같이 개선하면 장애 발생 전 탐지가 가능하다.

**개선 1: 경고 임계치 추가**

```bash
WARN_THRESHOLD=$((MEMORY_LIMIT * 80 / 100))  # MEMORY_LIMIT의 80%
if [ "$MEM" -gt "$WARN_THRESHOLD" ]; then
    echo "[$TIMESTAMP] [WARNING] Memory approaching limit: ${MEM}MB / ${MEMORY_LIMIT}MB" >> $LOG_FILE
fi
```

→ 한계치 도달 전 80% 시점에 경고를 남겨 사전 대응 가능

**개선 2: 메모리 증가 속도(Rate) 계산**

```bash
PREV_MEM=$(tail -2 $LOG_FILE | head -1 | grep -oP 'MEM:\K\d+')
RATE=$((MEM - PREV_MEM))
echo "[$TIMESTAMP] ... MEM:${MEM}MB RATE:+${RATE}MB/2s ..." >> $LOG_FILE
```

→ 단순 현재값이 아닌 증가 속도를 기록하면 "언제 한계에 도달할지" 예측 가능

**개선 3: 알림 연동**

```bash
if [ "$MEM" -gt "$WARN_THRESHOLD" ]; then
    curl -s -X POST "https://hooks.slack.com/..." \
      -d "{\"text\": \"[WARN] Memory ${MEM}MB / ${MEMORY_LIMIT}MB\"}"
fi
```

→ Slack/이메일 알림으로 담당자가 즉시 인지 가능

---

## Q9. 3가지 장애 중 실제 서비스 환경에서 가장 치명적인 것은 무엇이며, 근본 예방 방법은?

**가장 치명적인 장애: Deadlock**

OOM과 CPU Spike는 프로세스가 종료되며 명확한 에러 로그를 남긴다. 모니터링 시스템이 프로세스 종료를 즉시 감지하고 자동 재시작이 가능하다.

반면 Deadlock은 프로세스가 살아있는 것처럼 보이면서 아무것도 처리하지 않는다. 헬스체크가 "프로세스가 실행 중"이라고 잘못 판단하고, 사용자는 응답이 없는데 서버는 정상처럼 보이는 상황이 발생한다. 자동화된 탐지와 복구가 가장 어려운 유형이다.

**근본 예방 방법:**

| 방법 | 설명 |
|------|------|
| Lock Ordering | 모든 스레드가 Lock을 항상 같은 순서로 획득 (A→B 순서를 전역적으로 강제) |
| Timeout Lock | 일정 시간 내 Lock 획득 실패 시 보유 자원을 해제하고 재시도 |
| Deadlock Detection | 주기적으로 자원 할당 그래프에서 순환을 탐지하는 알고리즘 적용 |
| Lock-free 구조 | 가능한 경우 Lock 대신 CAS(Compare-And-Swap) 같은 원자적 연산 사용 |

---

## Q10. OOM과 Deadlock이 동시에 발생했다면 어떤 순서로 트러블슈팅하겠는가?

**우선순위: OOM 먼저**

**판단 근거:**

OOM은 메모리가 고갈되며 시스템 전체에 영향을 준다. RAM이 부족해지면 다른 정상 프로세스들도 메모리를 할당받지 못해 연쇄 장애가 발생한다. 시스템 전체가 불안정한 상태에서 Deadlock을 진단하려 해도 진단 도구 자체가 정상 작동하지 않을 수 있다.

반면 Deadlock은 해당 프로세스만 멈춰있고 다른 프로세스에는 영향을 주지 않는다.

**트러블슈팅 순서:**

```
1. OOM 프로세스를 kill하여 메모리 확보 → 시스템 안정화
2. 시스템 안정화 확인 (free -h로 메모리 여유 확인)
3. OOM 원인 로그 확보 (agent_app.log의 SELF-TERMINATED 로그)
4. Deadlock 프로세스 kill (kill PID)
5. 각각 원인 분석 및 재발 방지 조치
```

원칙: 시스템 전체 안정성을 먼저 확보한 뒤, 개별 서비스 장애를 순서대로 처리한다.

---

## Q11. 소스 코드를 직접 수정할 수 있다면 각 장애 유형별로 어떤 코드 레벨 개선을 하겠는가?

### OOM (메모리 누수)

```python
# 현재 (문제): 데이터를 계속 누적
def process_data():
    data = load_large_data()  # 메모리 할당
    # 처리 후 해제하지 않음

# 개선: 사용 후 즉시 해제
def process_data():
    data = load_large_data()
    result = calculate(data)
    del data  # 명시적 해제
    gc.collect()  # 가비지 컬렉션 강제 실행
    return result

# 또는 컨텍스트 매니저 사용
def process_data():
    with memory_managed_context() as data:
        return calculate(data)
    # with 블록 종료 시 자동 해제
```

### CPU Spike

```python
# 현재 (문제): 제한 없이 부하 생성
def generate_load():
    while True:
        heavy_computation()

# 개선: 슬립으로 CPU 양보 + 부하 상한 제한
def generate_load():
    while True:
        heavy_computation()
        time.sleep(0.01)  # 다른 프로세스에 CPU 양보
        if get_cpu_usage() > THRESHOLD:
            time.sleep(1)  # 임계치 초과 시 쿨다운
```

### Deadlock

```python
# 현재 (문제): Lock 획득 순서가 스레드마다 다름
# Thread-1: lock_A → lock_B
# Thread-2: lock_B → lock_A

# 개선 1: Lock Ordering (전역적으로 순서 통일)
LOCK_ORDER = [lock_A, lock_B]  # 항상 A → B 순서

def thread_function():
    for lock in LOCK_ORDER:  # 정해진 순서대로만 획득
        lock.acquire()

# 개선 2: Timeout Lock
def thread_function():
    if not lock_A.acquire(timeout=5):  # 5초 내 획득 실패 시
        return  # 보유 자원 해제 후 포기
    if not lock_B.acquire(timeout=5):
        lock_A.release()  # 보유 자원 해제
        return
```

---

## Q12. 다시 이 미션을 처음부터 수행한다면 어떤 점을 다르게 접근하겠는가?


**1. 초기 MEMORY_LIMIT을 512MB로 설정하여 앱 동작을 충분히 관측한다**

MEMORY_LIMIT=256으로 시작하니 앱이 30초 만에 종료되어 monitor.sh가 데이터를 충분히 수집하지 못했다. 처음부터 512MB로 설정하여 앱이 오래 실행되는 상태에서 전체 동작 패턴을 먼저 파악했을 것이다.

**2. 각 테스트 전 로그를 초기화하는 습관을 들인다**

```bash
> /var/log/agent-app/monitor.log
> /var/log/agent-app/agent_app.log
```

이전 테스트의 로그가 섞이면 분석이 어렵다. 매 테스트 시작 전 로그를 초기화하는 것을 체크리스트로 만든다.


**3. 각 케이스별 Before 데이터를 반드시 먼저 백업한 후 After로 넘어간다**

이번에는 Before 데이터 일부를 수동으로 복구해야 했다. 다음에는 Before 실행 후 즉시 백업하는 것을 원칙으로 삼는다.
