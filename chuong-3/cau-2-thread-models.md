CHƯƠNG 3 - CÂU 2
Đề bài:
So sánh hai cách cài đặt luồng: (a) hoàn toàn ở mức người dùng (user-level threads) và (b) kết hợp luồng ở mức người dùng với lightweight process (LWP) ở mức kernel. Phân tích ưu, nhược điểm của từng cách về chi phí chuyển ngữ cảnh, khả năng phong tỏa và độ phức tạp triển khai.

BÀI GIẢI:
Phần 1: Kiến thức nền tảng về Thread Models
A. Tổng quan về Thread Implementation
Có 3 mô hình chính để implement threads:
┌────────────────────────────────────────────────────┐
│  1. User-Level Threads (ULT)                      │
│     - Threads quản lý hoàn toàn ở user space     │
│     - Kernel không biết sự tồn tại của threads   │
│                                                    │
│  2. Kernel-Level Threads (KLT)                    │
│     - Threads quản lý bởi kernel                 │
│     - Kernel scheduling cho từng thread          │
│                                                    │
│  3. Hybrid Model (ULT + LWP)                      │
│     - Kết hợp ưu điểm của cả hai                 │
│     - User threads map lên kernel LWPs           │
└────────────────────────────────────────────────────┘

Phần 2: User-Level Threads (ULT)
A. Kiến trúc User-Level Threads
Định nghĩa:
User-Level Threads (ULT) là threads được tạo và quản lý hoàn toàn bởi thư viện runtime ở user space, không cần sự tham gia của kernel.
Kiến trúc:
┌────────────────────────────────────────────────────┐
│              USER SPACE                            │
│  ┌──────────────────────────────────────────┐     │
│  │         User Application                 │     │
│  │  [T1]  [T2]  [T3]  [T4]  [T5]           │     │
│  │   │     │     │     │     │              │     │
│  └───┼─────┼─────┼─────┼─────┼──────────────┘     │
│      │     │     │     │     │                     │
│  ┌───┴─────┴─────┴─────┴─────┴──────────────┐     │
│  │   Thread Library / Runtime Scheduler     │     │
│  │   - Thread table                         │     │
│  │   - User-level scheduler                 │     │
│  │   - Context switch handler               │     │
│  └──────────────────────────────────────────┘     │
│                    │                               │
│                    │ (System calls)                │
├────────────────────┼───────────────────────────────┤
│              KERNEL SPACE                          │
│  ┌─────────────────┴──────────────────────────┐   │
│  │        Single Kernel Thread / Process      │   │
│  │     (Kernel chỉ thấy 1 process)            │   │
│  └────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘

Đặc điểm:
- Kernel chỉ nhìn thấy 1 process
- Tất cả threads (T1-T5) invisible với kernel
- Thread scheduling ở user space
Cách hoạt động:
c// Thread Library maintains thread table
typedef struct {
    thread_id_t id;
    void* stack_pointer;      // SP khi thread bị preempt
    thread_state_t state;     // RUNNING, READY, BLOCKED
    int priority;
} UserThread;

UserThread thread_table[MAX_THREADS];
UserThread* current_thread;

// User-level scheduler
void schedule_next_thread() {
    // Chọn thread tiếp theo (round-robin, priority, etc.)
    UserThread* next = select_next_ready_thread();
    
    if (next != current_thread) {
        // Context switch ở user space
        context_switch(current_thread, next);
    }
}

// Context switch không cần kernel
void context_switch(UserThread* old, UserThread* new) {
    // 1. Lưu context của old thread
    save_registers(old->stack_pointer);
    
    // 2. Chuyển sang context của new thread
    restore_registers(new->stack_pointer);
    
    // 3. Update current thread
    current_thread = new;
    
    // Không có system call! (Very fast!)
}
```

**Ví dụ thư viện ULT:**
- **GNU Pth** (Portable Threads)
- **Green Threads** (Java cũ, Ruby)
- **Fibers** (Windows, Ruby)
- **Goroutines** (Go - hybrid nhưng có ULT characteristics)

---

**B. Ưu điểm của User-Level Threads**

**1. Chi phí chuyển ngữ cảnh cực thấp (Ultra-low Context Switch Cost)**
```
User-Level Thread Context Switch:
┌─────────────────────────────────────────┐
│ 1. Save registers to user stack        │  ← No kernel involved
│ 2. Load registers from new thread stack│  ← Pure user-space operation
│ 3. Jump to new thread                   │  
└─────────────────────────────────────────┘

Chi phí: 
- 10-50 nanoseconds (0.01-0.05 microseconds)
- Chỉ là function call overhead!
- No system call, no kernel trap, no TLB flush
```

**So sánh với Kernel Threads:**
```
Benchmark: 1 million context switches

User-Level Threads:
Time: 10-50 milliseconds
Cost per switch: 10-50 ns

Kernel-Level Threads:
Time: 1,000-10,000 milliseconds
Cost per switch: 1-10 μs

→ ULT nhanh hơn 100-1000x!
```

**Tại sao nhanh?**
```
ULT Context Switch:
1. SAVE:  push registers to stack        (5 cycles)
2. UPDATE: thread_table[current].sp      (2 cycles)
3. LOAD:  pop registers from new stack   (5 cycles)
4. JUMP:  jmp to new thread PC           (1 cycle)
Total: ~13 CPU cycles

KLT Context Switch:
1. User → Kernel mode transition         (50 cycles)
2. SAVE all registers + FPU state        (100 cycles)
3. Update kernel thread table            (20 cycles)
4. TLB flush (if different process)      (500 cycles)
5. Scheduler decision                    (100 cycles)
6. LOAD new thread state                 (100 cycles)
7. Kernel → User mode transition         (50 cycles)
Total: ~920 CPU cycles (70x slower!)
Ví dụ code:
c// ULT context switch (pseudo-assembly)
void user_context_switch(Thread* old, Thread* new) {
    // Inline assembly - no system call
    asm volatile(
        "pushq %%rbx\n"      // Save registers
        "pushq %%rbp\n"
        "pushq %%r12\n"
        "pushq %%r13\n"
        "pushq %%r14\n"
        "pushq %%r15\n"
        "movq %%rsp, %0\n"   // Save stack pointer
        "movq %1, %%rsp\n"   // Load new stack pointer
        "popq %%r15\n"       // Restore registers
        "popq %%r14\n"
        "popq %%r13\n"
        "popq %%r12\n"
        "popq %%rbp\n"
        "popq %%rbx\n"
        : "=m"(old->sp)
        : "m"(new->sp)
    );
    // Total: ~20 instructions, <50ns
}

2. Linh hoạt trong scheduling (Flexible Scheduling)
c// Application có thể implement custom scheduler
void application_specific_scheduler() {
    // Ví dụ: Game engine với priority-based scheduling
    
    if (high_priority_render_thread->ready) {
        schedule(high_priority_render_thread);
    } else if (medium_priority_physics_thread->ready) {
        schedule(medium_priority_physics_thread);
    } else {
        schedule(low_priority_background_thread);
    }
    
    // Kernel không can thiệp!
}

// Hoặc: Cooperative scheduling cho real-time systems
void cooperative_yield() {
    current_thread->state = READY;
    schedule_next_thread();  // Explicit yield
}
```

**Lợi ích:**
```
✅ Tùy chỉnh policy theo application needs
✅ Real-time scheduling cho embedded systems
✅ Priority inversion avoidance
✅ Domain-specific optimizations

3. Không cần kernel support (Portable)
c// Chạy trên bất kỳ OS nào
#ifdef LINUX
    // Linux-specific syscalls
#elif defined(WINDOWS)
    // Windows-specific APIs
#else
    // Generic POSIX
#endif

// Thread library tự xử lý
// Không phụ thuộc vào kernel thread support
```

**Ý nghĩa:**
```
✅ Portability: Chạy trên OS cũ không có thread support
✅ Consistency: Behavior giống nhau trên mọi platform
✅ Independence: Không bị giới hạn bởi kernel policy

4. Có thể tạo hàng triệu threads (Massive Scalability)
c// Kernel threads: Limited by kernel resources
// Typical limit: 1,000 - 10,000 threads

// User threads: Only limited by memory
Thread threads[1000000];  // 1 million threads!

for (int i = 0; i < 1000000; i++) {
    threads[i] = create_user_thread(task_function);
}

// Chi phí mỗi thread:
// - Stack: 4 KB (customizable, có thể 1 KB)
// - Thread control block: 100 bytes
// Total: ~4.1 KB per thread
// 1M threads = 4.1 GB RAM (acceptable!)

// Kernel threads would crash the system!
```

**Use case:**
```
✅ Erlang/Elixir: Millions of lightweight processes
✅ Go: Goroutines (100,000+ concurrent)
✅ Web servers: Handle millions of connections

C. Nhược điểm của User-Level Threads
1. Vấn đề phong tỏa (Blocking Problem) - NGHIÊM TRỌNG
Vấn đề:
c// Thread T1 gọi blocking system call
void* thread1_function(void* arg) {
    // T1 đọc file
    int fd = open("large_file.txt", O_RDONLY);
    char buffer[4096];
    
    read(fd, buffer, 4096);  // BLOCKING SYSTEM CALL!
    
    // Kernel block toàn bộ process
    // → T2, T3, T4 cũng bị block! ❌
}

void* thread2_function(void* arg) {
    // T2 muốn chạy nhưng KHÔNG THỂ
    // Vì kernel đã block cả process
    compute_something();  // Not executed!
}
```

**Visualization:**
```
TIME →
─────────────────────────────────────────────

Process (Kernel view: single execution unit)
├─ T1: RUNNING ──→ read() ──→ BLOCKED (5 seconds)
│
├─ T2: READY ──→ CANNOT RUN ❌ (waiting for T1)
│
├─ T3: READY ──→ CANNOT RUN ❌
│
└─ T4: READY ──→ CANNOT RUN ❌

Problem: 1 thread blocks → ALL threads block!
```

**Các syscalls gây blocking:**
```
❌ read(), write() (file I/O)
❌ recv(), send() (network I/O)
❌ accept() (waiting for connections)
❌ sleep() (timed delay)
❌ wait(), waitpid() (process synchronization)
❌ semaphore operations
❌ page faults (memory access)

→ Hầu hết I/O operations đều blocking!
```

**Hậu quả:**
```
Web Server với ULT:

Thread 1: Đọc request từ socket (block 100ms)
Thread 2: Muốn xử lý request khác (CANNOT! ❌)
Thread 3: Muốn gửi response (CANNOT! ❌)
Thread 4: Idle (CANNOT run! ❌)

Result: Server chỉ xử lý 1 request tại 1 thời điểm
Throughput: 10 requests/second (TERRIBLE!)
Giải pháp workaround (Không hoàn hảo):
c// 1. Non-blocking I/O + polling
fcntl(fd, F_SETFL, O_NONBLOCK);

while (1) {
    int n = read(fd, buffer, size);
    if (n > 0) {
        // Data available
        break;
    } else if (errno == EAGAIN) {
        // No data, yield to other threads
        user_thread_yield();  // Let other threads run
        continue;
    }
}

// Problem: Busy-waiting wastes CPU ❌


// 2. Wrapper around blocking calls
int wrapped_read(int fd, void* buf, size_t count) {
    // Check if data available (select/poll)
    fd_set readfds;
    FD_ZERO(&readfds);
    FD_SET(fd, &readfds);
    
    struct timeval timeout = {0, 0};  // Non-blocking
    int ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
    
    if (ret > 0) {
        return read(fd, buf, count);
    } else {
        // No data, yield
        user_thread_yield();
        return -1;  // Try again later
    }
}

// Problem: Complex, still not perfect ❌
```

---

**2. Không tận dụng được đa lõi (No Multicore Parallelism)**

**Vấn đề:**
```
4-Core CPU:
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Core 1│ │Core 2│ │Core 3│ │Core 4│
└───┬──┘ └──────┘ └──────┘ └──────┘
    │
    │ ← Only 1 core used!
    │
┌───┴────────────────────────────────┐
│  Process (with 10 user threads)    │
│  [T1] [T2] [T3] ... [T10]          │
│  Kernel sees: 1 execution unit     │
└────────────────────────────────────┘

Result:
- Core 1: 100% utilization
- Core 2: 0% (idle) ❌
- Core 3: 0% (idle) ❌
- Core 4: 0% (idle) ❌
- Total CPU utilization: 25%
```

**So sánh với Kernel Threads:**
```
Kernel Threads on 4-Core CPU:
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Core 1│ │Core 2│ │Core 3│ │Core 4│
│  T1  │ │  T2  │ │  T3  │ │  T4  │
└──────┘ └──────┘ └──────┘ └──────┘
   ✅       ✅       ✅       ✅
   
All cores used! (100% utilization)
```

**Benchmark:**
```
Matrix Multiplication (1000x1000):

User-Level Threads (10 threads):
- Uses 1 core
- Time: 10 seconds
- Wasted: 3 cores (75% waste)

Kernel-Level Threads (10 threads):
- Uses 4 cores (2-3 threads per core)
- Time: 3 seconds (3.3x faster!)
- Utilization: 100%

3. Không có preemption từ kernel
Vấn đề:
c// Selfish thread monopolizes CPU
void* bad_thread(void* arg) {
    while (1) {
        // Compute-intensive, không yield
        compute_forever();
        
        // Không gọi user_thread_yield()!
        // Kernel không can thiệp vì chỉ thấy 1 process đang chạy
    }
}

void* good_thread(void* arg) {
    // Muốn chạy nhưng KHÔNG BAO GIỜ được CPU!
    important_work();  // Never executes ❌
}

// Problem: Requires cooperative scheduling
// → 1 misbehaving thread stalls everything
```

**Kernel threads khác:**
```
Kernel preemptive scheduling:

Thread T1: Running... → TIMER INTERRUPT → PREEMPTED
                                          ↓
Thread T2:                            SCHEDULED
                                          ↓
                                      Running...

→ Fairness guaranteed by kernel

4. Phức tạp trong việc handle signals
c// Signal handler gây confusion
void signal_handler(int signo) {
    // Signal được deliver đến process
    // Nhưng thread nào nên handle?
    
    // Thread library phải:
    // 1. Intercept signal
    // 2. Determine which thread to deliver to
    // 3. Context switch to that thread
    // → Complex!
}

// Với kernel threads:
// - Signal trực tiếp đến specific thread
// - Simpler!
```

---

**D. Bảng tóm tắt User-Level Threads**

| Tiêu chí | Đánh giá | Chi tiết |
|----------|----------|----------|
| **Context Switch** | ✅✅✅ Rất tốt | 10-50 ns (100-1000x nhanh hơn KLT) |
| **Blocking I/O** | ❌❌ Rất kém | 1 thread block → tất cả block |
| **Multicore** | ❌❌ Rất kém | Chỉ dùng 1 core |
| **Scheduling Flexibility** | ✅✅ Tốt | Custom scheduler, domain-specific |
| **Scalability** | ✅✅✅ Rất tốt | Hàng triệu threads |
| **Portability** | ✅✅ Tốt | Không cần kernel support |
| **Complexity** | ✅ Đơn giản | Thread library tương đối simple |
| **Preemption** | ❌ Kém | Requires cooperative scheduling |
| **Signal Handling** | ❌ Kém | Complex to implement |

---

#### **Phần 3: Hybrid Model - ULT + LWP (Lightweight Process)**

**A. Kiến trúc Hybrid Model**

**Định nghĩa:**
Hybrid model kết hợp user-level threads với kernel-level lightweight processes (LWP). Multiple user threads được multiplex lên multiple kernel LWPs.

**Kiến trúc:**
```
┌────────────────────────────────────────────────────┐
│              USER SPACE                            │
│                                                    │
│    User Threads (Many-to-Many mapping)            │
│    [UT1] [UT2] [UT3] [UT4] [UT5] [UT6] [UT7]     │
│      │     │     │     │     │     │     │        │
│      └─────┼─────┼─────┘     └─────┼─────┘        │
│            │     │                 │               │
│  ┌─────────┴─────┴─────┬───────────┴───────┐      │
│  │   Thread Library    │                    │      │
│  │   - Scheduler       │                    │      │
│  │   - Multiplexer     │                    │      │
│  └─────────┬───────────┴───────────┬────────┘      │
│            │                       │                │
├────────────┼───────────────────────┼────────────────┤
│      KERNEL SPACE                                   │
│            │                       │                │
│      ┌─────▼─────┐           ┌────▼──────┐         │
│      │   LWP 1   │           │   LWP 2   │         │
│      │(Kernel    │           │(Kernel    │         │
│      │ Thread)   │           │ Thread)   │         │
│      └─────┬─────┘           └────┬──────┘         │
│            │                      │                 │
│      ┌─────▼──────────────────────▼─────┐          │
│      │      Kernel Scheduler            │          │
│      └──────────────────────────────────┘          │
└────────────────────────────────────────────────────┘

Đặc điểm:
- N user threads map to M LWPs (N > M)
- LWPs là kernel threads
- Thread library multiplexes UTs lên LWPs
```

**Mapping models:**
```
Model 1: Many-to-One (Pure ULT)
[UT1] [UT2] [UT3] [UT4]
   └─────┴─────┴─────┘
         │
      [LWP 1]
      
Problem: Blocking, no multicore


Model 2: One-to-One (Pure KLT)
[UT1]  [UT2]  [UT3]  [UT4]
  │      │      │      │
[LWP1] [LWP2] [LWP3] [LWP4]

Problem: Too many kernel threads, high overhead


Model 3: Many-to-Many (Hybrid) ✅
[UT1] [UT2] [UT3] [UT4] [UT5] [UT6]
   └────┬────┬────┘  └────┬────┘
      [LWP1]       [LWP2]
      
Balance: Flexibility + Performance

B. Cách hoạt động của Hybrid Model
Thread binding và scheduling:
c// Thread library maintains binding
typedef struct {
    UserThread* user_thread;
    LWP* bound_lwp;          // LWP đang chạy thread này
    binding_type_t type;      // BOUND hoặc UNBOUND
} ThreadBinding;

// Unbound threads: Dynamically assigned to available LWP
// Bound threads: Permanently bound to specific LWP

void schedule_user_thread(UserThread* ut) {
    if (ut->type == BOUND) {
        // Run on specific LWP
        run_on_lwp(ut, ut->bound_lwp);
    } else {
        // Find available LWP
        LWP* available = find_idle_lwp();
        if (available) {
            bind_temporarily(ut, available);
            run_on_lwp(ut, available);
        } else {
            // All LWPs busy, enqueue
            enqueue_ready(ut);
        }
    }
}
Xử lý blocking:
c// Thread T1 calls blocking syscall
void* thread1(void* arg) {
    char buffer[4096];
    
    // Blocking read
    int n = read(fd, buffer, 4096);
    
    // What happens internally:
    // 1. Thread library detects syscall
    // 2. LWP blocks in kernel (OK!)
    // 3. Thread library assigns T2 to another LWP
    // 4. T2 continues running ✅
}

// LWP pool management
void handle_blocking_call(UserThread* ut, LWP* lwp) {
    // 1. LWP will block, unbind current thread
    unbind(ut, lwp);
    
    // 2. Find another ready thread
    UserThread* next = get_next_ready_thread();
    
    // 3. Assign to different LWP
    LWP* free_lwp = find_free_lwp();
    if (free_lwp) {
        bind(next, free_lwp);
        resume(next);
    } else {
        // Create new LWP if needed
        LWP* new_lwp = create_lwp();
        bind(next, new_lwp);
        resume(next);
    }
    
    // 4. Original syscall proceeds on blocked LWP
}
```

**Visualization:**
```
TIME →
───────────────────────────────────────────────

LWP 1: [UT1 running] → [UT1 blocks on I/O]
                            ↓
                      (LWP 1 blocks in kernel)
                      
LWP 2: [UT2 running] → [UT2 continues] → [UT3 scheduled]
       (unaffected!)       ✅               ✅

Result: 
- UT1 blocks: Only LWP 1 affected
- UT2, UT3: Continue on LWP 2
- No process-wide blocking! ✅

C. Ưu điểm của Hybrid Model
1. Giải quyết vấn đề blocking (Solves Blocking Problem)
c// Thread pool với LWPs
void handle_client_requests() {
    // 10 user threads, 3 LWPs
    UserThread threads[10];
    LWP lwps[3];
    
    // Thread 1 blocks on I/O
    threads[0].read_from_socket();  // LWP 1 blocks
    
    // Threads 2-10 continue on LWP 2, 3
    threads[1].process_request();   // LWP 2 (active)
    threads[2].process_request();   // LWP 3 (active)
    threads[3].enqueued();          // Waits for free LWP
    
    // When LWP 1 unblocks:
    // → Thread 1 completes
    // → Thread 3 scheduled on LWP 1
}

Result:
✅ Blocking I/O không stall toàn bộ application
✅ Multiple I/O operations đồng thời
✅ Better concurrency
```

**Benchmark:**
```
Web Server handling 1000 requests:

User-Level Threads only:
- 1 blocking I/O → all threads stalled
- Throughput: 10 req/sec
- Latency: 100ms average

Hybrid Model (10 UT, 3 LWP):
- 3 concurrent I/O operations
- Throughput: 300 req/sec (30x better!)
- Latency: 30ms average
```

---

**2. Tận dụng đa lõi (Multicore Utilization)**
```
4-Core CPU:
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Core 1│ │Core 2│ │Core 3│ │Core 4│
│ LWP1 │ │ LWP2 │ │ LWP3 │ │ LWP4 │
│  UT1 │ │  UT3 │ │  UT5 │ │  UT7 │
│  UT2 │ │  UT4 │ │  UT6 │ │  UT8 │
└──────┘ └──────┘ └──────┘ └──────┘
   ✅       ✅       ✅       ✅

All cores active!
- 4 LWPs scheduled on 4 cores
- 8 user threads multiplexed
- CPU utilization: 100%
```

**Performance:**
```
Parallel Matrix Computation:

Pure ULT (20 threads):
- 1 core utilized
- Time: 40 seconds

Hybrid (20 UT, 4 LWP):
- 4 cores utilized
- Time: 10 seconds (4x faster!)
- Speedup matches hardware parallelism
```

---

**3. Cân bằng giữa performance và overhead**
```
Context Switch Analysis:

User-level switch (UT1 → UT2 on same LWP):
- Cost: 50 ns (ultra-fast!)
- Frequency: High (100,000/sec)
- Total overhead: 5ms/sec

Kernel-level switch (LWP1 → LWP2):
- Cost: 5 μs (moderate)
- Frequency: Low (100/sec)
- Total overhead: 0.5ms/sec

Combined overhead: 5.5ms/sec (0.55%)
→ Negligible!

Compare with pure KLT (all switches kernel-level):
- Cost: 5 μs
- Frequency: 100,000/sec
- Total overhead: 500ms/sec (50%!)

4. Flexibility trong resource management
c// Application có thể điều chỉnh số LWPs
void set_concurrency_level(int num_lwps) {
    // Increase LWPs for I/O-bound workload
    if (io_intensive) {
        create_lwps(20);  // Nhiều LWPs
        // → Handle nhiều blocking calls đồng thời
    }
    
    // Decrease LWPs for CPU-bound workload
    if (cpu_intensive) {
        create_lwps(num_cores);  // Chỉ cần bằng số cores
        // → Minimize overhead
    }
}

// Dynamic adjustment
void monitor_and_adjust() {
    if (blocked_lwp_ratio > 0.5) {
        // Nhiều LWPs bị block
        create_additional_lwp();
    } else if (idle_lwp_ratio > 0.3) {
        // LWPs thừa
        destroy_idle_lwp();
    }
}

D. Nhược điểm của Hybrid Model
1. Độ phức tạp triển khai (Implementation Complexity) - CAO
c// Thread library phải quản lý:

1. User-level scheduler
   - Round-robin, priority-based, etc.
   - Ready queue, blocked queue

2. LWP pool management
   - Tạo/hủy LWPs động
   - Bind/unbind threads

3. Blocking detection
   - Intercept syscalls
   - Wrapper functions

4. Two-level synchronization
   - User-level locks (fast path)
   - Kernel-level locks (when needed)

5. Signal handling
   - Route signals đúng thread
   - Coordinate với LWPs

// Hàng ngàn dòng code phức tạp!
Code example:
c// Simplified thread library structure
typedef struct ThreadLibrary {
    // User thread management
    UserThread* ready_queue[MAX_THREADS];
    UserThread* blocked_queue[MAX_THREADS];
    UserThread* current_thread;
    
    // LWP pool
    LWP* lwp_pool[MAX_LWPS];
    int num_lwps;
    int max_lwps;
    
    // Binding table
    ThreadBinding bindings[MAX_THREADS];
    
    // Synchronization
    mutex_t library_lock;
    
    // Scheduler state
    SchedulerPolicy policy;
    int time_slice;
    
    // Signal handling
    SignalHandler signal_handlers[MAX_SIGNALS];
    
    // Statistics
    uint64_t context_switches;
    uint64_t lwp_blocks;
    uint64_t migrations;
} ThreadLibrary;

// Implementation: 10,000+ lines of code
// Bugs: Race conditions, deadlocks, memory leaks
// Maintenance: Nightmare!

2. Chi phí coordination (Coordination Overhead)
c// Thread library và kernel phải coordinate

// Example: Thread migration
void migrate_thread(UserThread* ut, LWP* old_lwp, LWP* new_lwp) {
    // 1. User-level operation
    lock(&library_lock);
    unbind(ut, old_lwp);
    bind(ut, new_lwp);
    unlock(&library_lock);
    
    // 2. Kernel-level operation
    // LWP switches context (kernel involved)
    // → Both user and kernel overhead!
    
    // Total cost: ~2-3 μs (slower than pure ULT)
}

// Frequent migrations → Overhead accumulates
```

**Overhead analysis:**
```
Scenario: High thread churn

Pure ULT:
- Context switches: 50 ns each
- 100,000 switches/sec
- Overhead: 5 ms

Hybrid:
- User-level: 50 ns
- Occasional LWP switch: 5 μs
- Migration: 3 μs
- Total overhead: ~20 ms (4x higher)

Still better than pure KLT (500ms), but not as good as ULT

3. Vấn đề priority inversion
c// LWP scheduling vs User thread scheduling mismatch

Scenario:
- High-priority UT1 on LWP1
- Low-priority UT2 on LWP2
- LWP2 has higher kernel priority than LWP1

Problem:
Kernel schedules LWP2 (low priority thread runs!)
→ UT1 (high priority) starves!

Timeline:
LWP1 (UT1 high-pri): [wait] [wait] [wait] [run]
LWP2 (UT2 low-pri):  [run] [run] [run] [wait]
                      ↑ WRONG!

Solution: Complex coordination between schedulers
→ Adds overhead and complexity

4. Không đảm bảo fairness hoàn hảo
c// Unbound threads có thể bị "starvation"

Scenario: 10 UT, 2 LWP

LWP 1: [UT1] [UT1] [UT1] [UT1] ...  (monopolizes)
LWP 2: [UT2] [UT2] [UT2] [UT2] ...  (monopolizes)

UT3-UT10: Waiting... waiting... ❌

Reason:
- UT1, UT2 CPU-bound, never yield
- LWPs continuously run them
- Other threads starve

Solution: User-level preemption
→ Requires timer signals, complex
```

---

**E. Bảng tóm tắt Hybrid Model**

| Tiêu chí | Đánh giá | Chi tiết |
|----------|----------|----------|
| **Context Switch** | ✅✅ Tốt | User-level: 50ns, Kernel: 5μs |
| **Blocking I/O** | ✅✅✅ Rất tốt | LWP blocks, others continue |
| **Multicore** | ✅✅✅ Rất tốt | LWPs run on multiple cores |
| **Scalability** | ✅✅ Tốt | Many UTs on few LWPs |
| **Implementation** | ❌❌ Rất phức tạp | Two-level management |
| **Coordination Overhead** | ⚠️ Trung bình | User-kernel coordination |
| **Priority Inversion** | ⚠️ Có thể xảy ra | Two-level scheduling conflict |
| **Fairness** | ⚠️ Không hoàn hảo | Requires careful tuning |
| **Debugging** | ❌ Khó | Complex interactions |

---

#### **Phần 4: So sánh chi tiết hai mô hình**

**A. Chi phí chuyển ngữ cảnh (Context Switch Cost)**

**Bảng so sánh:**

| Loại Switch | User-Level Threads | Hybrid Model |
|-------------|-------------------|--------------|
| **User thread → User thread (same LWP)** | 50 ns | 50 ns |
| **User thread → User thread (different LWP)** | N/A | 3 μs (migration) |
| **LWP → LWP** | N/A | 5 μs |
| **Typical workload** | 50 ns avg | 200-500 ns avg |

**Phân tích:**
```
User-Level Threads:
✅ Luôn nhanh (50ns)
✅ Predictable performance
✅ No kernel involvement

Hybrid Model:
✅ Fast trong LWP (50ns)
⚠️ Slower khi migrate (3μs)
⚠️ Variable performance
⚠️ Kernel sometimes involved

Verdict: ULT thắng về context switch speed
```

---

**B. Khả năng phong tỏa (Blocking Behavior)**

**So sánh chi tiết:**
```
User-Level Threads:

Thread T1: read(fd) → BLOCKS
    ↓
Kernel blocks entire process
    ↓
T2, T3, T4: CANNOT RUN ❌
    ↓
Application STALLED until I/O completes

Impact: CRITICAL FAILURE


Hybrid Model:

Thread T1: read(fd) → BLOCKS
    ↓
LWP1 blocks in kernel
    ↓
Thread library: Schedule T2 on LWP2 ✅
    ↓
T2, T3, T4: CONTINUE RUNNING
    ↓
Application remains RESPONSIVE

Impact: HANDLED GRACEFULLY
```

**Benchmark:**
```
Test: 100 threads, 50 blocking I/O calls

User-Level Threads:
- Sequential execution (1 at a time)
- Time: 50 × 100ms = 5,000 ms
- Throughput: 20 ops/sec

Hybrid Model (10 LWPs):
- 10 concurrent I/O operations
- Time: 50 × 100ms / 10 = 500 ms
- Throughput: 200 ops/sec (10x better!)

Verdict: Hybrid thắng hoàn toàn về blocking

C. Độ phức tạp triển khai (Implementation Complexity)
User-Level Threads:
c// Thread library code (simplified)

struct UserThread {
    void* stack;
    void* sp;
    int state;
};

void schedule() {
    // Simple round-robin
    current = (current + 1) % num_threads;
    context_switch(&threads[current]);
}

void context_switch(UserThread* t) {
    // Save/restore registers
    // 50 lines of assembly
}

// Total implementation: ~1,000 lines
// Complexity: LOW ✅
Hybrid Model:
c// Thread library (partial structure)

// User thread management: 500 lines
// LWP pool management: 800 lines
// Binding management: 400 lines
// Blocking detection: 600 lines
// Two-level sync: 1,000 lines
// Signal handling: 700 lines
// Priority management: 500 lines
// Debug/stats: 500 lines

// Total: 5,000+ lines
// Complexity: VERY HIGH ❌

// Plus kernel modifications:
// - LWP implementation: 2,000 lines
// - Coordination interface: 500 lines

// Grand total: 7,500+ lines
// Maintenance nightmare!
```

**Verdict:**
```
User-Level: Simple, 1-2 weeks to implement
Hybrid: Complex, 3-6 months to implement correctly

→ ULT thắng về độ đơn giản
```

---

**D. Tận dụng đa lõi (Multicore Utilization)**
```
4-Core CPU, CPU-bound workload:

User-Level Threads (10 threads):
Core utilization: [100%, 0%, 0%, 0%]
Total: 25%
Speedup: 1x ❌

Hybrid Model (10 UT, 4 LWP):
Core utilization: [100%, 100%, 100%, 100%]
Total: 100%
Speedup: 4x ✅

Verdict: Hybrid thắng về multicore
```

---

**E. Bảng so sánh tổng hợp**

| Tiêu chí | User-Level Threads | Hybrid Model | Winner |
|----------|-------------------|--------------|---------|
| **Context switch cost** | 50 ns ✅✅✅ | 200-500 ns ✅✅ | ULT |
| **Blocking behavior** | Process-wide ❌❌ | Per-LWP ✅✅✅ | **Hybrid** |
| **Multicore usage** | Single core ❌❌ | All cores ✅✅✅ | **Hybrid** |
| **Implementation complexity** | Simple ✅✅✅ | Complex ❌❌ | ULT |
| **Scalability** | Millions ✅✅✅ | Thousands ✅✅ | ULT |
| **Predictability** | High ✅✅ | Medium ⚠️ | ULT |
| **I/O intensive workload** | Poor ❌❌ | Excellent ✅✅✅ | **Hybrid** |
| **CPU intensive workload** | Poor (1 core) ❌ | Excellent ✅✅✅ | **Hybrid** |
| **Debugging** | Easy ✅✅ | Hard ❌ | ULT |
| **Portability** | High ✅✅ | Medium ⚠️ | ULT |

---

#### **Phần 5: Kết luận và khuyến nghị**

**Khi nên dùng User-Level Threads:**
```
✅ Use cases:
1. Embedded systems (no kernel thread support)
2. Extremely high thread count (millions)
3. Minimal context switch overhead critical
4. Pure compute-bound, no I/O
5. Legacy OS support needed

❌ Avoid when:
1. Blocking I/O operations
2. Multi-core CPU available
3. Mixed I/O and compute workload
```

**Khi nên dùng Hybrid Model:**
```
✅ Use cases:
1. General-purpose applications
2. Web servers (I/O + compute)
3. Multi-core systems
4. Mixed workload (I/O + CPU)
5. Need both performance and concurrency

❌ Avoid when:
1. Simple applications
2. Development resources limited
3. Ultra-low latency required (every nanosecond counts)
```

**Thực tế hiện đại:**
```
Most modern systems use Hybrid or Kernel-Level:

- Java: JVM uses OS threads (kernel-level)
        + Virtual threads (Project Loom) - hybrid-like

- Go: Goroutines = Hybrid model (M:N mapping)
      - Millions of goroutines on thousands of OS threads

- Rust: Tokio = User-level (async/await)
        + OS threads for blocking ops

- Erlang: Millions of lightweight processes (hybrid-like)

→ Trend: Hybrid approaches winning!
```

---

**Trade-off summary:**
```
User-Level Threads:
+ Cực nhanh (context switch)
+ Cực đơn giản (implement)
+ Scalability cao
- Blocking = disaster
- Single core only
- No true parallelism

Hybrid Model:
+ Giải quyết blocking
+ Tận dụng multicore
+ Balanced performance
- Phức tạp triển khai
- Overhead cao hơn ULT
- Khó debug
