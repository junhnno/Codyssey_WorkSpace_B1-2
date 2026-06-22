# Codyssey_WorkSpace_B1-2

# Linux 프로세스 트러블슈팅 - GitHub Issue 리포트 3건

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
sed -i 's/MEMORY_LIMIT=256/MEMORY_LIMIT=512/' ~/.bash_profile
source ~/.bash_profile
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
- **Watchdog 동작 원리**: 애플리케이션 내부의 Watchdog은 CPU 사용률을 지속적으로 모니터링하다가 설정된 임계치를 초과하면 SIGTERM 신호를 프로세스에 보내 강제 종료한다. 이는 특정 프로세스의 CPU 독점으로 인해 시스템 전체가 느려지는 것을 방지하기 위한 보호 정책이다.
- **OS 원리**: CPU는 여러 프로세스가 시분할(Time Sharing) 방식으로 공유하는 자원이다. 하나의 프로세스가 CPU를 과점유하면 다른 프로세스들이 CPU를 할당받지 못해 응답 지연 또는 타임아웃이 발생한다.

---

## 4. Workaround & Verification (조치 및 검증)

### 조치 내용

`~/.bash_profile`의 `CPU_MAX_OCCUPY` 값을 90% → 50%로 하향 조정하여 CPU 부하가 임계치를 초과하지 않도록 제한했다.

```bash
sed -i 's/CPU_MAX_OCCUPY=90/CPU_MAX_OCCUPY=50/' ~/.bash_profile
source ~/.bash_profile
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

→ 50% 도달 후 cooldown 반복하며 프로세스 종료 없이 정상 동작

### 근본 해결 제안

임시 조치(CPU_MAX_OCCUPY 하향)는 부하 한도를 낮추는 것이므로 성능 저하를 감수해야 한다. 근본적인 해결을 위해서는 CpuWorker 내부의 부하 생성 로직을 최적화하여 불필요한 CPU 소비를 줄이는 리팩토링이 필요하다.

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

### PID 존재 확인 (프로세스 살아있음)

```bash
$ ps -ef | grep agent-leak-app-x86 | grep -v grep
agent  13781  24  0 09:37 pts/1  00:00:00 /home/agent-leak-app-x86
agent  13782 13781  0 09:37 pts/1  00:00:00 /home/agent-leak-app-x86
```

### 스레드 상태 확인 (CPU 사용 없음)

```bash
$ ps -L -p 13781
  PID   LWP TTY          TIME CMD
13781 13781 pts/1    00:00:00 agent-leak-app-
```

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

이는 Deadlock 발생의 4대 조건을 모두 충족한다.

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
sed -i 's/MULTI_THREAD_ENABLE=true/MULTI_THREAD_ENABLE=false/' ~/.bash_profile
source ~/.bash_profile
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

→ 데드락 없이 스케줄러가 순차적으로 태스크를 완료하며 정상 동작

### 근본 해결 제안

멀티스레드 비활성화는 동시성을 포기하는 임시 조치이다. 근본적인 해결을 위해서는 Lock 획득 순서를 전역적으로 통일하거나(Lock Ordering), 타임아웃 기반 Lock 획득 로직을 도입하여 Deadlock을 방지해야 한다.
