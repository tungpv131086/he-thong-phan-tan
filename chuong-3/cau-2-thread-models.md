# Câu 1: Thread vs Process - Context Switching

> **Chương:** 3 - Tiến trình và Ảo hóa  
> **Độ khó:** ⭐⭐⭐ (Trung bình)  
> **Thời gian đọc:** ~20 phút

---

## 📋 Mục lục

- [Đề bài](#đề-bài)
- [Phần 1: Định nghĩa Thread và Process](#phần-1-định-nghĩa-thread-và-process)
- [Phần 2: Context Switch Analysis](#phần-2-context-switch-analysis)
- [Phần 3: Multicore Performance](#phần-3-multicore-performance)
- [Phần 4: Use Cases](#phần-4-use-cases)
- [Tóm tắt](#tóm-tắt)

---

## 📋 Đề bài

Một hệ thống có hai mô hình thực thi:

**Mô hình A: Multi-process**
- Mỗi request được xử lý bởi một **process** riêng biệt
- Context switch giữa processes: **50 µs**

**Mô hình B: Multi-threaded**
- Mỗi request được xử lý bởi một **thread** trong cùng process
- Context switch giữa threads: **5 µs**

**Yêu cầu:**

1. Giải thích sự khác biệt giữa **thread** và **process** về:
   - Memory space
   - Resource sharing
   - Communication overhead

2. Tính toán và so sánh **overhead** của context switching:
   - Với 10,000 switches/second
   - Impact lên throughput

3. Phân tích khi nào nên dùng **multi-process** vs **multi-threaded** trong hệ thống phân tán, đặc biệt trên **multicore processors**

---

## 💡 Bài giải

### Phần 1: Định nghĩa Thread và Process

#### A. Process (Tiến trình)
```
╔═══════════════════════════════════════════════╗
║              PROCESS STRUCTURE                ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  ┌─────────────────────────────────────────┐ ║
║  │   Process Control Block (PCB)           │ ║
║  │   - Process ID (PID)                    │ ║
║  │   - Program counter                     │ ║
║  │   - CPU registers                       │ ║
║  │   - Memory management info              │ ║
║  │   - I/O status                          │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
║  ┌─────────────────────────────────────────┐ ║
║  │   MEMORY SPACE (isolated)               │ ║
║  │                                         │ ║
║  │   ┌─────────────────────────────────┐  │ ║
║  │   │  Stack (local variables)        │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Heap (dynamic allocation)      │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Data (global variables)        │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Code (executable instructions) │  │ ║
║  │   └─────────────────────────────────┘  │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
║  ┌─────────────────────────────────────────┐ ║
║  │   RESOURCES (exclusive)                 │ ║
║  │   - File descriptors                    │ ║
║  │   - Network sockets                     │ ║
║  │   - Locks                               │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
╚═══════════════════════════════════════════════╝

Characteristics:
✅ Isolated memory (cannot access other process memory)
✅ Independent resources
✅ Heavyweight (large overhead to create)
✅ Safe (crash doesn't affect other processes)
⚠️ Slow communication (IPC required)
```

#### B. Thread (Luồng)
```
╔═══════════════════════════════════════════════╗
║           MULTI-THREADED PROCESS              ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  SHARED RESOURCES:                            ║
║  ┌─────────────────────────────────────────┐ ║
║  │   Process Control Block (PCB)           │ ║
║  │   - Process ID (shared)                 │ ║
║  │   - Memory management (shared)          │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
║  ┌─────────────────────────────────────────┐ ║
║  │   SHARED MEMORY SPACE                   │ ║
║  │                                         │ ║
║  │   ┌─────────────────────────────────┐  │ ║
║  │   │  Stack Thread 1 (private)       │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Stack Thread 2 (private)       │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Stack Thread 3 (private)       │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Heap (SHARED) ✅               │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Data (SHARED) ✅               │  │ ║
║  │   ├─────────────────────────────────┤  │ ║
║  │   │  Code (SHARED) ✅               │  │ ║
║  │   └─────────────────────────────────┘  │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
║  PER-THREAD STATE:                            ║
║  ┌───────────┬───────────┬───────────┐       ║
║  │ Thread 1  │ Thread 2  │ Thread 3  │       ║
║  │ - TID     │ - TID     │ - TID     │       ║
║  │ - PC      │ - PC      │ - PC      │       ║
║  │ - Regs    │ - Regs    │ - Regs    │       ║
║  │ - Stack   │ - Stack   │ - Stack   │       ║
║  └───────────┴───────────┴───────────┘       ║
║                                               ║
╚═══════════════════════════════════════════════╝

Characteristics:
✅ Shared memory (fast communication)
✅ Shared resources
✅ Lightweight (low overhead to create)
⚠️ Unsafe (one thread crash can kill process)
⚠️ Need synchronization (race conditions)
```

#### C. Key Differences
```
╔═══════════════════════════════════════════════════╗
║        PROCESS vs THREAD COMPARISON               ║
╠═══════════════════════════════════════════════════╣
║                                                   ║
║  Aspect          │ Process         │ Thread       ║
║ ═════════════════╪═════════════════╪═════════════ ║
║                                                   ║
║  Memory Space    │ Separate ✅     │ Shared ✅    ║
║                  │ 4GB+ per proc   │ 2MB+ total   ║
║                                                   ║
║  Creation Time   │ 1-5 ms ⚠️       │ 50-100 µs ✅ ║
║                                                   ║
║  Context Switch  │ 50 µs ⚠️        │ 5 µs ✅      ║
║                                                   ║
║  Communication   │ IPC (pipe, MQ)  │ Shared mem   ║
║                  │ Slow ⚠️         │ Fast ✅      ║
║                                                   ║
║  Overhead        │ High ⚠️         │ Low ✅       ║
║                  │ (TLB flush)     │ (cache hit)  ║
║                                                   ║
║  Isolation       │ Strong ✅       │ Weak ⚠️      ║
║                  │ (protected)     │ (shared)     ║
║                                                   ║
║  Failure Impact  │ Isolated ✅     │ Cascading ❌ ║
║                                                   ║
║  Scalability     │ Limited ⚠️      │ Better ✅    ║
║                  │ (OS limits)     │ (thousands)  ║
║                                                   ║
╚═══════════════════════════════════════════════════╝
```

---

### Phần 2: Context Switch Analysis

#### A. Context Switch Cost Breakdown

**Process Context Switch (50 µs):**
```
PROCESS CONTEXT SWITCH:
══════════════════════════════════════════════

1. Save current process state (5 µs)
   ├─ Save CPU registers
   ├─ Save program counter
   ├─ Save stack pointer
   └─ Update PCB

2. Select next process (2 µs)
   ├─ Scheduler decision
   └─ Priority calculation

3. Memory management switch (30 µs) ⚠️
   ├─ Save current page table
   ├─ Load new page table
   ├─ TLB flush (Translation Lookaside Buffer)
   └─ Cache pollution (data no longer relevant)

4. Load next process state (8 µs)
   ├─ Restore CPU registers
   ├─ Restore program counter
   ├─ Update CPU mode
   └─ Resume execution

5. Overhead (5 µs)
   ├─ Kernel transition
   └─ Interrupt handling

Total: ~50 µs ⚠️

Main cost: Memory management (60% of time) ❌
```

**Thread Context Switch (5 µs):**
```
THREAD CONTEXT SWITCH:
══════════════════════════════════════════════

1. Save current thread state (1 µs)
   ├─ Save CPU registers
   ├─ Save program counter
   └─ Save stack pointer

2. Select next thread (0.5 µs)
   ├─ Scheduler decision (simpler)
   └─ Priority calculation

3. Memory management (0 µs) ✅
   ├─ NO page table switch (same process!)
   ├─ NO TLB flush
   └─ Cache remains valid ✅

4. Load next thread state (2 µs)
   ├─ Restore CPU registers
   ├─ Restore program counter
   └─ Resume execution

5. Overhead (1.5 µs)
   └─ Context switch logic

Total: ~5 µs ✅

Main benefit: No memory management overhead! ✅
```

#### B. Performance Calculation

**Scenario: 10,000 context switches/second**
```python
# Constants
switches_per_second = 10_000

# Process context switch
process_switch_time = 50e-6  # 50 microseconds
thread_switch_time = 5e-6    # 5 microseconds

# Calculate CPU time spent on context switching
process_overhead = switches_per_second * process_switch_time
thread_overhead = switches_per_second * thread_switch_time

# Calculate as percentage of CPU time
process_overhead_pct = process_overhead * 100
thread_overhead_pct = thread_overhead * 100

print("CPU time on context switching:")
print(f"Multi-process: {process_overhead:.3f}s = {process_overhead_pct:.1f}%")
print(f"Multi-threaded: {thread_overhead:.3f}s = {thread_overhead_pct:.1f}%")
print(f"Difference: {(process_overhead - thread_overhead):.3f}s")
print(f"Thread is {process_overhead/thread_overhead:.1f}× faster")
```

**Results:**
```
╔═══════════════════════════════════════════════════╗
║     CONTEXT SWITCHING OVERHEAD (10K/sec)          ║
╠═══════════════════════════════════════════════════╣
║                                                   ║
║  Model          │ Time/switch │ Total   │ CPU %  ║
║ ════════════════╪═════════════╪═════════╪═══════ ║
║                                                   ║
║  Multi-process  │    50 µs    │ 0.50s   │  50%   ║
║                 │             │         │  ⚠️    ║
║                                                   ║
║  Multi-threaded │     5 µs    │ 0.05s   │   5%   ║
║                 │             │         │  ✅    ║
║                                                   ║
║  Savings        │    45 µs    │ 0.45s   │  45%   ║
║                 │             │         │  ✅✅  ║
║                                                   ║
╚═══════════════════════════════════════════════════╝

Impact on throughput:
─────────────────────────────────────────────

Multi-process:
- Available CPU: 50% (rest is context switching)
- Max throughput: 50% of theoretical max ⚠️

Multi-threaded:
- Available CPU: 95% (only 5% context switching)
- Max throughput: 95% of theoretical max ✅

Improvement: 1.9× throughput with threads! 🚀
```

#### C. Scalability Analysis
```
╔═══════════════════════════════════════════════════╗
║        CONTEXT SWITCH RATE vs OVERHEAD            ║
╠═══════════════════════════════════════════════════╣
║                                                   ║
║  Switches/sec │ Process  │ Thread   │ Difference ║
║               │ Overhead │ Overhead │            ║
║ ══════════════╪══════════╪══════════╪═══════════ ║
║               │          │          │            ║
║     1,000     │    5%    │   0.5%   │    4.5%    ║
║    10,000     │   50%    │    5%    │   45% ✅   ║
║   100,000     │  500%❗  │   50%    │  450% ✅✅  ║
║ 1,000,000     │ 5000%❗  │  500% ⚠️ │ 4500%      ║
║               │          │          │            ║
╚═══════════════════════════════════════════════════╝

Observations:
- At 10K switches/sec: Process model saturated ❌
- At 100K switches/sec: Both models unusable
- Thread model scales 10× better ✅

Recommendation:
- Keep context switches < 10K/sec for processes
- Keep context switches < 100K/sec for threads
```

---

### Phần 3: Multicore Performance

#### A. Multi-Process on Multicore
```
╔═══════════════════════════════════════════════╗
║      MULTI-PROCESS ON 4-CORE CPU              ║
╠═══════════════════════════════════════════════╣
║                                               ║
║   Core 0      Core 1      Core 2      Core 3 ║
║  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐  ║
║  │ P1   │   │ P2   │   │ P3   │   │ P4   │  ║
║  │      │   │      │   │      │   │      │  ║
║  │ Own  │   │ Own  │   │ Own  │   │ Own  │  ║
║  │ Mem  │   │ Mem  │   │ Mem  │   │ Mem  │  ║
║  └──────┘   └──────┘   └──────┘   └──────┘  ║
║                                               ║
║  ┌─────────────────────────────────────────┐ ║
║  │         SHARED NOTHING                  │ ║
║  │  - Each core runs independently        │ ║
║  │  - No cache sharing                    │ ║
║  │  - No lock contention ✅               │ ║
║  │  - Communication via IPC (slow) ⚠️     │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
╚═══════════════════════════════════════════════╝

Advantages:
✅ True parallelism (4× throughput)
✅ No synchronization overhead
✅ Fault isolation (one crash doesn't affect others)
✅ Linear scaling (4 cores = 4× performance)

Disadvantages:
⚠️ High memory usage (4× memory)
⚠️ Slow inter-process communication
⚠️ Complex data sharing (requires serialization)
```

#### B. Multi-Threaded on Multicore
```
╔═══════════════════════════════════════════════╗
║     MULTI-THREADED ON 4-CORE CPU              ║
╠═══════════════════════════════════════════════╣
║                                               ║
║   Core 0      Core 1      Core 2      Core 3 ║
║  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐  ║
║  │ T1   │   │ T2   │   │ T3   │   │ T4   │  ║
║  │      │   │      │   │      │   │      │  ║
║  └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘  ║
║     │          │          │          │       ║
║     └──────────┴──────────┴──────────┘       ║
║                    │                          ║
║  ┌─────────────────▼────────────────────────┐║
║  │         SHARED MEMORY                    │║
║  │  - Heap (data structures)                │║
║  │  - Global variables                      │║
║  │  - Fast communication ✅                 │║
║  │  - Need locks (contention) ⚠️           │║
║  └──────────────────────────────────────────┘║
║                                               ║
╚═══════════════════════════════════════════════╝

Advantages:
✅ Low memory usage (shared heap)
✅ Fast communication (shared memory)
✅ Easy data sharing (no serialization)
✅ Lower overhead per thread

Disadvantages:
⚠️ Lock contention (mutex, semaphore)
⚠️ Race conditions (hard to debug)
⚠️ Cache coherency overhead
❌ One thread crash kills all threads
```

#### C. Performance Comparison
```python
# Simulation: Process 4 requests on 4 cores

import time

# Multi-process model
def multiprocess_benchmark():
    """
    Each process independent, no synchronization
    """
    processes = 4
    request_time = 100e-3  # 100ms per request
    
    # Parallel execution (no overhead)
    total_time = request_time
    
    # Communication overhead (if needed)
    ipc_overhead = 5e-3 * 3  # 5ms × 3 IPC calls
    
    return total_time + ipc_overhead

# Multi-threaded model
def multithread_benchmark():
    """
    Threads share data, need synchronization
    """
    threads = 4
    request_time = 100e-3
    
    # Parallel execution
    total_time = request_time
    
    # Lock contention overhead
    lock_overhead = 0.5e-3 * 10  # 0.5ms × 10 lock acquisitions
    
    # Cache coherency overhead
    cache_overhead = 1e-3
    
    return total_time + lock_overhead + cache_overhead

# Calculate
mp_time = multiprocess_benchmark()
mt_time = multithread_benchmark()

print(f"Multi-process: {mp_time*1000:.2f}ms")
print(f"Multi-threaded: {mt_time*1000:.2f}ms")
print(f"Winner: {'Thread' if mt_time < mp_time else 'Process'}")
```

**Results:**
```
╔═══════════════════════════════════════════════╗
║       4-CORE PERFORMANCE COMPARISON           ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Metric              │ Process │ Thread      ║
║ ═════════════════════╪═════════╪════════════ ║
║                                               ║
║  Request time        │ 100ms   │ 100ms       ║
║  IPC overhead        │  15ms   │   0ms ✅    ║
║  Lock overhead       │   0ms   │   5ms ⚠️    ║
║  Cache overhead      │   0ms   │   1ms ⚠️    ║
║ ─────────────────────┼─────────┼────────────ᅵ║
║  Total               │ 115ms   │ 106ms ✅    ║
║                                               ║
║  Throughput (req/s)  │  34.8   │  37.7 ✅    ║
║  Speedup (vs single) │  3.5×   │  3.8× ✅    ║
║                                               ║
╚═══════════════════════════════════════════════╝

Conclusion: Threads slightly faster (8%) ✅
```

---

### Phần 4: Use Cases

#### A. When to Use Multi-Process
```
╔═══════════════════════════════════════════════╗
║         USE MULTI-PROCESS WHEN:               ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  1. Strong Isolation Required ✅              ║
║     - Untrusted code execution               ║
║     - Security-critical applications         ║
║     - Example: Web browser tabs              ║
║                                               ║
║  2. Fault Tolerance Critical ✅               ║
║     - One task failure shouldn't kill others ║
║     - Example: Nginx worker processes        ║
║                                               ║
║  3. Different Programming Languages ✅        ║
║     - Microservices (Python + Go + Java)     ║
║     - Example: Service-oriented architecture ║
║                                               ║
║  4. Independent Tasks ✅                      ║
║     - No/minimal communication needed        ║
║     - Example: Video encoding pipeline       ║
║                                               ║
║  5. Legacy Code Integration ✅                ║
║     - Cannot modify existing processes       ║
║     - Example: Enterprise system integration ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

**Example: Web Server (Nginx)**
```c
// Nginx: Multi-process architecture
void master_process() {
    for (int i = 0; i < num_workers; i++) {
        pid_t pid = fork();
        
        if (pid == 0) {
            // Child process: Worker
            worker_process();
            exit(0);
        }
    }
    
    // Master: Monitor workers
    while (1) {
        pid_t dead_worker = wait(NULL);
        // Restart dead worker
        fork_new_worker();
    }
}

Benefits:
✅ One worker crash doesn't affect others
✅ Master can reload config without downtime
✅ Security: Workers run with low privileges
```

#### B. When to Use Multi-Threaded
```
╔═══════════════════════════════════════════════╗
║        USE MULTI-THREADED WHEN:               ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  1. High Data Sharing ✅                      ║
║     - Frequent communication between tasks   ║
║     - Example: Database query engine         ║
║                                               ║
║  2. Low Latency Required ✅                   ║
║     - Fast context switching needed          ║
║     - Example: Real-time trading systems     ║
║                                               ║
║  3. Memory Constrained ✅                     ║
║     - Limited RAM available                  ║
║     - Example: Embedded systems              ║
║                                               ║
║  4. Many Concurrent Tasks ✅                  ║
║     - Thousands of simultaneous requests     ║
║     - Example: Web server (Apache)           ║
║                                               ║
║  5. Shared State Required ✅                  ║
║     - Global data structures                 ║
║     - Example: In-memory cache (Redis)       ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

**Example: Web Server (Apache)**
```java
// Apache: Multi-threaded worker
class ApacheWorker extends Thread {
    private ConnectionPool pool;
    private Cache cache;  // Shared!
    
    public void run() {
        while (true) {
            Request req = pool.accept();
            
            // Check shared cache (fast!)
            if (cache.has(req.url)) {
                send(cache.get(req.url));
            } else {
                Response res = processRequest(req);
                cache.put(req.url, res);
                send(res);
            }
        }
    }
}

Benefits:
✅ All threads share cache (memory efficient)
✅ Fast communication (shared memory)
✅ Low overhead (5µs context switch)
```

#### C. Hybrid Approach
```
╔═══════════════════════════════════════════════╗
║          HYBRID: BEST OF BOTH WORLDS          ║
╠═══════════════════════════════════════════════╣
║                                               ║
║   Master Process                              ║
║         │                                     ║
║   ┌─────┼─────┬─────┬─────┐                  ║
║   │     │     │     │     │                   ║
║   P1    P2    P3    P4    P5 (Worker procs)  ║
║   │     │     │     │     │                   ║
║  ┌┴┐   ┌┴┐   ┌┴┐   ┌┴┐   ┌┴┐                 ║
║  │T1   │T1   │T1   │T1   │T1                 ║
║  │T2   │T2   │T2   │T2   │T2  (Threads)      ║
║  │T3   │T3   │T3   │T3   │T3                 ║
║  └─┘   └─┘   └─┘   └─┘   └─┘                 ║
║                                               ║
║  Benefits:                                    ║
║  ✅ Fault isolation (between processes)      ║
║  ✅ Fast communication (within process)      ║
║  ✅ Scalability (threads) + Stability (procs)║
║                                               ║
╚═══════════════════════════════════════════════╝

Examples:
- Chrome: Multi-process (tabs) + multi-thread (rendering)
- PostgreSQL: Multi-process (connections) + worker threads
- Node.js Cluster: Multi-process + event loop
```

---

## 📊 Tóm tắt

### Key Points

- ✅ **Process**: Isolated memory, 50µs context switch, fault-tolerant
- ✅ **Thread**: Shared memory, 5µs context switch, fast communication
- ✅ **Context switch overhead**: Thread 10× faster than process
- ✅ **10K switches/sec**: 50% CPU (process) vs 5% CPU (thread)
- ✅ **Multicore**: Both scale linearly, threads slightly faster

### Decision Matrix
```
╔═══════════════════════════════════════════════╗
║           WHEN TO USE WHICH MODEL             ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Requirement        │ Multi-Process │ Thread ║
║ ════════════════════╪═══════════════╪═══════ ║
║                                               ║
║  Fault isolation    │      ✅       │   ❌   ║
║  Fast communication │      ❌       │   ✅   ║
║  Low memory usage   │      ❌       │   ✅   ║
║  Security           │      ✅       │   ⚠️   ║
║  Low latency        │      ❌       │   ✅   ║
║  Scalability        │      ⚠️       │   ✅   ║
║  Debugging ease     │      ✅       │   ❌   ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

### Recommendations

| Scenario | Choice | Reason |
|----------|--------|--------|
| Web server (high traffic) | **Thread** ✅ | Low overhead, shared cache |
| Microservices | **Process** ✅ | Isolation, different languages |
| Real-time system | **Thread** ✅ | Low latency (5µs switch) |
| Batch processing | **Process** ✅ | Independent tasks |
| In-memory database | **Thread** ✅ | Shared data structures |
| Critical infrastructure | **Hybrid** ✅ | Best of both worlds |

---

## 🔗 Tài liệu tham khảo

### Books
- **Operating System Concepts** - Silberschatz (Chapter 4)
- **Modern Operating Systems** - Tanenbaum (Chapter 2)

### Papers
- **Scheduler Activations** - Anderson et al., 1991
- **The Performance of µ-Kernel-Based Systems** - Liedtke, 1995

---

## 🧭 Navigation

**[⬅️ Chương 2](../chuong-2/README.md)** | **[📚 Quay lại Chương 3](./README.md)** | **[➡️ Câu 2: Thread Models](./cau-2-thread-models.md)**

---

*Cập nhật lần cuối: 11/12/2025*