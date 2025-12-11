CHƯƠNG 3 - CÂU 1
Đề bài:
Hãy định nghĩa khái niệm luồng (thread) và nêu sự khác biệt cơ bản giữa luồng và tiến trình (process) trong hệ điều hành. Giải thích tại sao việc chia tiến trình thành nhiều luồng có thể cải thiện hiệu năng trên máy tính đa bộ vi xử lý.

BÀI GIẢI:
Phần 1: Định nghĩa Luồng (Thread)
A. Khái niệm Thread
Định nghĩa:
Thread (luồng) là đơn vị thực thi nhỏ nhất trong một tiến trình, là một chuỗi các lệnh (sequence of instructions) có thể được CPU thực thi độc lập.
Đặc điểm chính:
Thread bao gồm:
- Thread ID (định danh duy nhất)
- Program Counter (PC): Con trỏ lệnh đang thực thi
- Register Set: Tập thanh ghi CPU
- Stack: Vùng nhớ stack riêng cho local variables và function calls
Mối quan hệ với Process:
1 Process có thể chứa nhiều Threads
┌─────────────── PROCESS ───────────────┐
│                                        │
│  [Thread 1]  [Thread 2]  [Thread 3]   │
│      │            │            │       │
│   Stack 1     Stack 2     Stack 3     │
│      │            │            │       │
│  ┌────────────────────────────────┐   │
│  │    SHARED RESOURCES:           │   │
│  │  - Code (instructions)         │   │
│  │  - Data (global variables)     │   │
│  │  - Heap (dynamic memory)       │   │
│  │  - Files (open file handles)   │   │
│  └────────────────────────────────┘   │
└────────────────────────────────────────┘
Ví dụ thực tế:
Web Browser (1 process):
├─ Thread 1: Render UI (hiển thị giao diện)
├─ Thread 2: Download files (tải file)
├─ Thread 3: Execute JavaScript (chạy JS)
└─ Thread 4: Handle user input (xử lý click, typing)

Tất cả threads chia sẻ:
- Cùng bộ nhớ browser
- Cùng cookies, cache
- Cùng các tab đang mở

B. Định nghĩa Process (Tiến trình)
Định nghĩa:
Process (tiến trình) là một chương trình đang thực thi, là đơn vị phân bổ tài nguyên của hệ điều hành.
Đặc điểm:
Process bao gồm:
- Process ID (PID)
- Address Space (không gian địa chỉ riêng)
- Code segment (mã lệnh)
- Data segment (dữ liệu tĩnh)
- Heap (bộ nhớ động)
- Stack (cho mỗi thread)
- File descriptors (các file đang mở)
- I/O resources (devices, network connections)
Process isolation:
PROCESS A          PROCESS B
┌────────┐         ┌────────┐
│ Code   │         │ Code   │
│ Data   │         │ Data   │
│ Heap   │         │ Heap   │
│ Stack  │         │ Stack  │
└────────┘         └────────┘
     │                  │
     └──────────────────┘
     Isolated (không can thiệp nhau)

Phần 2: So sánh Thread vs Process
Bảng so sánh chi tiết:
Tiêu chíProcessThreadĐịnh nghĩaChương trình đang chạyĐơn vị thực thi trong processTài nguyênCó address space riêngChia sẻ address space với threads khácBộ nhớĐộc lập, isolatedShared memory (code, data, heap)StackMỗi process có stack riêngMỗi thread có stack riêngChi phí tạoCao (5-10ms)Thấp (0.1-1ms)Chi phí context switchCao (10-100μs)Thấp (1-10μs)Giao tiếpIPC (Inter-Process Communication) phức tạpShared memory (đơn giản)An toànCao (isolated)Thấp (race conditions)Crash impactChỉ ảnh hưởng process đóToàn bộ process crash

A. Sự khác biệt về Tài nguyên (Resource Ownership)
Process:
c// Process A
int global_var = 10;  // Riêng của Process A

void function_a() {
    global_var = 20;   // Chỉ Process A thấy
    printf("%d\n", global_var);  // Output: 20
}

// Process B
int global_var = 10;  // Riêng của Process B (khác Process A)

void function_b() {
    global_var = 30;   // Process A KHÔNG bị ảnh hưởng
    printf("%d\n", global_var);  // Output: 30
}
Thread:
c// Trong cùng 1 Process
int global_var = 10;  // CHIA SẺ cho tất cả threads

// Thread 1
void* thread1_function(void* arg) {
    global_var = 20;   // Thay đổi ảnh hưởng tất cả threads!
    printf("T1: %d\n", global_var);  // Output: 20
}

// Thread 2
void* thread2_function(void* arg) {
    printf("T2: %d\n", global_var);  // Output: 20 (bị Thread 1 thay đổi)
    global_var = 30;
}

// Thread 3 sẽ thấy global_var = 30
```

**Visualization:**
```
PROCESS A                      PROCESS B
┌─────────────────┐           ┌─────────────────┐
│ global_var = 20 │           │ global_var = 30 │
│                 │           │                 │
│ Thread 1        │           │ Thread 1        │
│ Thread 2        │           │                 │
└─────────────────┘           └─────────────────┘
      ↑                             ↑
 Isolated Memory            Isolated Memory
 (không thấy nhau)          (không thấy nhau)


SINGLE PROCESS với Multiple Threads
┌───────────────────────────────────┐
│     SHARED: global_var = 30       │
│  ┌──────┐  ┌──────┐  ┌──────┐    │
│  │ T1   │  │ T2   │  │ T3   │    │
│  │stack │  │stack │  │stack │    │
│  └──────┘  └──────┘  └──────┘    │
│     ↑          ↑          ↑       │
│     └──────────┴──────────┘       │
│       All see global_var          │
└───────────────────────────────────┘

B. Chi phí tạo (Creation Cost)
Tạo Process:
c// Fork một process mới (Linux)
pid_t pid = fork();

Các bước OS thực hiện:
1. Cấp phát PID mới
2. Copy toàn bộ address space của parent
   - Code segment (~1MB)
   - Data segment (~500KB)
   - Heap (~2MB)
   - Stack (~8MB)
   → Total: ~10-100MB cần copy!
3. Setup page tables (memory mapping)
4. Copy file descriptors
5. Setup scheduling info

Chi phí: ~5-10 milliseconds
Tạo Thread:
c// Tạo thread mới (POSIX)
pthread_t thread;
pthread_create(&thread, NULL, thread_function, NULL);

Các bước OS thực hiện:
1. Cấp phát Thread ID
2. Tạo stack riêng (~1-2MB)
3. Copy register context
4. Add vào scheduling queue

Không cần copy:
✓ Code (đã có trong process)
✓ Data (shared)
✓ Heap (shared)
✓ File descriptors (shared)

Chi phí: ~0.1-1 millisecond (nhanh hơn 10-100x!)
```

**Benchmark thực tế:**
```
Test: Tạo 1000 processes vs 1000 threads

Creating 1000 processes:
Time: 5,234 ms
Memory: 10 GB

Creating 1000 threads:
Time: 87 ms (60x faster!)
Memory: 2 GB (5x less!)
```

---

**C. Chi phí chuyển ngữ cảnh (Context Switch Cost)**

**Process Context Switch:**
```
OS phải:
1. Lưu toàn bộ trạng thái process hiện tại:
   - Program counter
   - Tất cả CPU registers (20-30 registers)
   - Stack pointer
   - Memory mapping (page tables)
   
2. Flush TLB (Translation Lookaside Buffer)
   → Cache miss khi process mới chạy
   
3. Load trạng thái process mới:
   - Restore registers
   - Switch page tables (change CR3 register)
   - Reload TLB
   
4. Flush CPU caches (L1, L2)
   → Process mới có data khác

Chi phí: 10-100 microseconds
```

**Thread Context Switch:**
```
OS chỉ cần:
1. Lưu trạng thái thread:
   - Program counter
   - CPU registers
   - Stack pointer
   
2. Load trạng thái thread mới
   
KHÔNG cần:
✓ Switch page tables (cùng address space)
✓ Flush TLB (vẫn valid)
✓ Flush caches nhiều (shared data)

Chi phí: 1-10 microseconds (nhanh hơn 10x!)
```

**Visualization:**
```
PROCESS CONTEXT SWITCH:
┌────────────────────────────────────┐
│ Process A running                  │
│ ├─ Save ALL registers             │
│ ├─ Save memory mapping             │
│ ├─ Flush TLB                       │
│ ├─ Flush Caches                    │  ← Expensive!
│ │                                   │
│ Switch Page Tables                 │  ← Very Expensive!
│ │                                   │
│ Load Process B                     │
│ ├─ Restore registers               │
│ ├─ Reload TLB (cache miss)         │
│ └─ Reload Caches (cache miss)      │
└────────────────────────────────────┘
Time: ~50 μs


THREAD CONTEXT SWITCH (same process):
┌────────────────────────────────────┐
│ Thread 1 running                   │
│ ├─ Save registers                  │
│ │                                   │
│ Switch to Thread 2 (same process)  │  ← Fast!
│ │                                   │
│ ├─ Restore registers               │
│ └─ Continue (TLB valid, cache hot) │
└────────────────────────────────────┘
Time: ~5 μs (10x faster!)

D. Giao tiếp (Communication)
Inter-Process Communication (IPC):
c// Process A và B cần giao tiếp

// Method 1: Pipe
int fd[2];
pipe(fd);  // Create pipe

if (fork() == 0) {  // Child process
    close(fd[0]);  // Close read end
    write(fd[1], "Hello", 5);  // Write to pipe
} else {  // Parent process
    close(fd[1]);  // Close write end
    char buf[10];
    read(fd[0], buf, 5);  // Read from pipe
}
// → Phức tạp, cần system calls

// Method 2: Shared Memory
int shmid = shmget(IPC_PRIVATE, 1024, IPC_CREAT);
char* shared = shmat(shmid, NULL, 0);
// → Phức tạp, cần setup

// Method 3: Message Queue
// Method 4: Sockets
// → Tất cả đều phức tạp và chậm!
Inter-Thread Communication:
c// Threads chia sẻ memory → Giao tiếp đơn giản!

int shared_data = 0;  // Global variable
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

// Thread 1
void* thread1(void* arg) {
    pthread_mutex_lock(&lock);
    shared_data = 42;  // Ghi trực tiếp!
    pthread_mutex_unlock(&lock);
}

// Thread 2
void* thread2(void* arg) {
    pthread_mutex_lock(&lock);
    printf("%d\n", shared_data);  // Đọc trực tiếp!
    pthread_mutex_unlock(&lock);
}

// → Đơn giản, nhanh (không cần system call)
```

**So sánh performance:**
```
Benchmark: 1 million message exchanges

Process (Pipe IPC):
Time: 2,500 ms
System calls: 2,000,000 (slow!)

Thread (Shared Memory):
Time: 50 ms (50x faster!)
System calls: 2,000,000 (only for locks)

E. An toàn và Độ ổn định (Safety & Stability)
Process:
c// Process isolation
Process A crashes:
- Segmentation fault
- Only Process A dies
- Process B, C, D continue running
- System remains stable ✅

Example:
Chrome Browser:
├─ Process per Tab
├─ Tab 1 crashes (malicious site)
├─ Other tabs unaffected ✅
└─ Browser continues working
Thread:
c// Shared memory → Dangerous!
Thread 1 bug:
- Writes to invalid pointer
- Corrupts shared memory
- ALL threads affected ❌
- ENTIRE process crashes

Example:
void* buggy_thread(void* arg) {
    int* ptr = NULL;
    *ptr = 42;  // Segfault!
    // → Process dies (all threads killed)
}

// Even Thread 2, 3, 4 (innocent) are killed!
Race conditions với threads:
cint counter = 0;  // Shared

// Thread 1
void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // Not atomic!
    }
}

// Thread 2
void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;
    }
}

// Expected: counter = 2,000,000
// Actual: counter = 1,234,567 (BUG! Race condition)

Assembly của counter++:
1. LOAD counter → register
2. ADD 1 → register
3. STORE register → counter

Thread 1 và 2 có thể interleave:
T1: LOAD (counter=0)
T2: LOAD (counter=0)
T1: ADD 1 (register=1)
T2: ADD 1 (register=1)
T1: STORE (counter=1)
T2: STORE (counter=1)  ← Overwrite! Lost increment!
```

---

#### **Phần 3: Tại sao Multithreading cải thiện hiệu năng trên đa lõi?**

**A. Kiến trúc CPU hiện đại**

**Single-core CPU (quá khứ):**
```
┌─────────────────┐
│   CPU Core      │
│                 │
│  [Processing    │
│   Unit]         │
└─────────────────┘

1 core chỉ chạy 1 instruction tại 1 thời điểm
Multithreading → Context switching (không tăng throughput)
```

**Multi-core CPU (hiện tại):**
```
┌──────────────────────────────────┐
│          CPU Chip                │
│  ┌──────┐  ┌──────┐  ┌──────┐   │
│  │Core 1│  │Core 2│  │Core 3│   │
│  │      │  │      │  │      │   │
│  └──────┘  └──────┘  └──────┘   │
│  ┌──────┐  ┌──────┐  ┌──────┐   │
│  │Core 4│  │Core 5│  │Core 6│   │
│  │      │  │      │  │      │   │
│  └──────┘  └──────┘  └──────┘   │
└──────────────────────────────────┘

6 cores → Có thể chạy 6 instructions ĐỒNG THỜI!
Multithreading → True parallelism!
```

**Ví dụ CPU phổ biến:**
```
Intel Core i7-13700K:
- 16 cores (8 Performance + 8 Efficiency)
- 24 threads (với Hyper-Threading)

AMD Ryzen 9 7950X:
- 16 cores
- 32 threads (với SMT)

Apple M2 Pro:
- 12 cores (8 Performance + 4 Efficiency)
```

---

**B. Parallel Execution (Thực thi song song)**

**Single-threaded program trên multi-core:**
```
Time →
Core 1: [████████████████████████████] (100% busy)
Core 2: [                            ] (idle)
Core 3: [                            ] (idle)
Core 4: [                            ] (idle)

Total work: 10 seconds
Utilization: 25% (1/4 cores)
Wasted potential: 75%
```

**Multi-threaded program trên multi-core:**
```
Time →
Core 1: [██████] Thread 1 (2.5s)
Core 2: [██████] Thread 2 (2.5s)
Core 3: [██████] Thread 3 (2.5s)
Core 4: [██████] Thread 4 (2.5s)

Total work: 2.5 seconds (4x faster!)
Utilization: 100%
Speedup: 4x (ideal)
Ví dụ code:
c// SINGLE-THREADED: Slow
void process_images(Image images[], int count) {
    for (int i = 0; i < count; i++) {
        resize_image(&images[i]);     // 1 second
        apply_filter(&images[i]);      // 1 second
        compress_image(&images[i]);    // 1 second
    }
}
// Process 100 images: 300 seconds (5 minutes)


// MULTI-THREADED: Fast
void* worker_thread(void* arg) {
    WorkUnit* unit = (WorkUnit*)arg;
    for (int i = unit->start; i < unit->end; i++) {
        resize_image(&images[i]);
        apply_filter(&images[i]);
        compress_image(&images[i]);
    }
}

void process_images_parallel(Image images[], int count) {
    pthread_t threads[4];
    WorkUnit units[4];
    
    // Chia công việc cho 4 threads
    for (int i = 0; i < 4; i++) {
        units[i].start = i * 25;      // 0, 25, 50, 75
        units[i].end = (i+1) * 25;    // 25, 50, 75, 100
        pthread_create(&threads[i], NULL, worker_thread, &units[i]);
    }
    
    // Đợi tất cả threads hoàn thành
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }
}
// Process 100 images: 75 seconds (1.25 minutes)
// Speedup: 4x faster!
```

---

**C. Overlapping I/O và Computation**

**Single-threaded program:**
```
Time →
0s    1s    2s    3s    4s    5s    6s    7s    8s
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ I/O │ CPU │ I/O │ CPU │ I/O │ CPU │ I/O │ CPU │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

Total: 8 seconds
CPU idle during I/O (50% wasted!)
```

**Multi-threaded program:**
```
Thread 1: I/O ─┐
              ↓ CPU ─┐
                     ↓ I/O ─┐
                            ↓ CPU

Thread 2:      I/O ─┐
                    ↓ CPU ─┐
                           ↓ I/O ─┐
                                  ↓ CPU

Time: 4 seconds (2x faster!)
CPU always busy (100% utilization)
Ví dụ thực tế: Web Server
c// SINGLE-THREADED: Blocking I/O
void handle_requests_sequential() {
    while (1) {
        int client_fd = accept(server_fd, ...);  // Wait for client
        
        read(client_fd, request, ...);           // Wait for data (I/O)
        process_request(request);                // CPU work
        write(client_fd, response, ...);         // Wait to send (I/O)
        
        close(client_fd);
    }
}
// Problem: Chỉ xử lý 1 client tại 1 thời điểm
// Throughput: ~10 requests/second


// MULTI-THREADED: Concurrent I/O
void* handle_client(void* arg) {
    int client_fd = *(int*)arg;
    
    read(client_fd, request, ...);      // Thread 1 đợi I/O
    process_request(request);           // Meanwhile Thread 2 xử lý
    write(client_fd, response, ...);    // Thread 3 gửi data
    
    close(client_fd);
}

void handle_requests_concurrent() {
    while (1) {
        int client_fd = accept(server_fd, ...);
        
        pthread_t thread;
        pthread_create(&thread, NULL, handle_client, &client_fd);
        pthread_detach(thread);  // Auto cleanup
    }
}
// Xử lý nhiều clients đồng thời
// Throughput: ~1000 requests/second (100x better!)
```

---

**D. Responsiveness (Khả năng phản hồi)**

**GUI Application không có threads:**
```
User clicks button
↓
Application: Đang xử lý... (3 seconds)
↓
UI FROZEN! (không phản hồi)
↓
User frustrated: "App bị treo?" ❌
```

**GUI Application với threads:**
```
Main Thread (UI Thread):
├─ Luôn lắng nghe user input
├─ Update UI (render)
└─ Responsive!

Background Thread:
└─ Xử lý công việc nặng
    ├─ Download file
    ├─ Compute heavy task
    └─ Database query

User clicks button
↓
UI Thread: Spawn background thread
↓
UI continues responding (smooth!) ✅
↓
Background thread hoàn thành → Update UI
Code example:
c// WITHOUT threads: UI frozen
void button_click_handler() {
    // Heavy computation (blocks UI)
    for (int i = 0; i < 1000000000; i++) {
        compute_something();
    }
    update_ui();  // UI frozen for 10 seconds ❌
}


// WITH threads: UI responsive
void button_click_handler() {
    // Spawn background thread
    pthread_t thread;
    pthread_create(&thread, NULL, background_task, NULL);
    
    // UI thread returns immediately (responsive!) ✅
}

void* background_task(void* arg) {
    for (int i = 0; i < 1000000000; i++) {
        compute_something();
    }
    
    // Update UI when done (via message queue)
    post_ui_update();
}
```

---

**E. Amdahl's Law - Giới hạn của Parallelism**

**Công thức:**
```
Speedup = 1 / [(1 - P) + P/N]

Where:
P = Phần có thể song song hóa (0 → 1)
N = Số cores/threads
1 - P = Phần tuần tự (không thể song song)
```

**Ví dụ tính toán:**
```
Program với 90% code có thể song song (P = 0.9)

1 core:  Speedup = 1.0x (baseline)
2 cores: Speedup = 1 / [0.1 + 0.9/2] = 1.82x
4 cores: Speedup = 1 / [0.1 + 0.9/4] = 3.08x
8 cores: Speedup = 1 / [0.1 + 0.9/8] = 4.71x
16 cores: Speedup = 1 / [0.1 + 0.9/16] = 6.40x
∞ cores: Speedup = 1 / 0.1 = 10x (maximum!)
```

**Visualization:**
```
P = 50% (Half parallel)
┌─────────────────────────────┐
│ Sequential │  Parallel      │
│    10%     │     90%        │
└─────────────────────────────┘
Max speedup: 10x

P = 90% (Mostly parallel)
┌──┬──────────────────────────┐
│S │      Parallel            │
│ 10%                         │
└──┴──────────────────────────┘
Max speedup: 10x

P = 99% (Almost all parallel)
┌┬───────────────────────────┐
│S│    Parallel              │
│1%                          │
└┴───────────────────────────┘
Max speedup: 100x
```

**Ý nghĩa:**
```
❗ Phần sequential (1-P) là bottleneck!
❗ Không thể tăng speed vô hạn bằng cách thêm cores
❗ Phải tối ưu cả phần sequential
```

---

**F. Các lợi ích khác**

**1. Resource Utilization:**
```
Modern CPU có:
- Multiple cores (4-32 cores)
- Hyper-Threading/SMT (2 threads per core)

Single-threaded app:
- Chỉ dùng 1 core
- Wasted 95% CPU capacity ❌

Multi-threaded app:
- Dùng tất cả cores
- Maximize hardware investment ✅
```

**2. Modularity:**
```
Program logic tách biệt:
- Thread 1: Handle network
- Thread 2: Process data
- Thread 3: Update database
- Thread 4: Generate reports

→ Clean separation of concerns
→ Easier to maintain
```

**3. Server Scalability:**
```
Web Server với threads:
- 1000 concurrent connections
- Each connection = 1 thread
- Can handle massive traffic

Without threads:
- Sequential processing
- Only 10-100 connections/second

Phần 4: Ví dụ thực tế
Example 1: Matrix Multiplication
c// Single-threaded: 10 seconds for 1000x1000 matrix
void matrix_multiply(int A[][], int B[][], int C[][], int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            C[i][j] = 0;
            for (int k = 0; k < N; k++) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
}


// Multi-threaded: 2.5 seconds on 4-core CPU
typedef struct {
    int start_row;
    int end_row;
} ThreadData;

void* matrix_multiply_thread(void* arg) {
    ThreadData* data = (ThreadData*)arg;
    
    for (int i = data->start_row; i < data->end_row; i++) {
        for (int j = 0; j < N; j++) {
            C[i][j] = 0;
            for (int k = 0; k < N; k++) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
}

void matrix_multiply_parallel(int A[][], int B[][], int C[][], int N) {
    pthread_t threads[4];
    ThreadData data[4];
    
    int rows_per_thread = N / 4;
    
    for (int i = 0; i < 4; i++) {
        data[i].start_row = i * rows_per_thread;
        data[i].end_row = (i + 1) * rows_per_thread;
        pthread_create(&threads[i], NULL, matrix_multiply_thread, &data[i]);
    }
    
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }
}

// Result: 4x speedup!

Example 2: Video Encoding
c// FFmpeg-style video encoder

// Single-threaded: Encode 1 frame at a time
for (int i = 0; i < num_frames; i++) {
    Frame* frame = read_frame(i);
    encode_frame(frame);          // 100ms per frame
    write_frame(frame);
}
// 1000 frames = 100 seconds


// Multi-threaded: Pipeline parallelism
Thread 1 (Reader):     Read frames from disk
                       ↓
Thread 2-5 (Encoders): Encode 4 frames in parallel
                       ↓
Thread 6 (Writer):     Write encoded frames

// 1000 frames = 25 seconds (4x faster!)

Tóm tắt:
Định nghĩa:

Thread: Đơn vị thực thi trong process, shared memory
Process: Chương trình đang chạy, isolated memory

Sự khác biệt chính:

Memory: Process isolated, Thread shared
Cost: Thread rẻ hơn (creation, context switch)
Communication: Thread dễ hơn (shared memory)
Safety: Process an toàn hơn (isolation)

Tại sao Multithreading cải thiện performance trên đa lõi:

True Parallelism: Nhiều cores chạy đồng thời
Better Resource Utilization: Tận dụng 100% CPU
Overlapping I/O and Computation: Không bị idle
Responsiveness: UI luôn smooth
Scalability: Xử lý nhiều tasks cùng lúc

Lưu ý:

Chỉ hiệu quả với đa lõi (multi-core)
Phải thiết kế code song song được (Amdahl's Law)
Cần quản lý synchronization (locks, semaphores)


