CHƯƠNG 3 - CÂU 4
Đề bài:
Hãy định nghĩa ảo hóa (virtualization) và nêu vai trò chính của kỹ thuật ảo hóa trong các hệ thống phân tán. Phân loại các hình thức ảo hóa dựa trên lớp giao diện (instruction-set virtualization vs. system-level virtualization):

Máy ảo tiến trình (process-level VM, ví dụ Java Runtime)
Giám sát máy ảo (hypervisor-based VM)
Mô tả cơ chế hoạt động và điểm khác biệt cơ bản giữa hai hình thức này.


BÀI GIẢI:
Phần 1: Định nghĩa và vai trò của Ảo hóa
A. Định nghĩa Ảo hóa (Virtualization)
Định nghĩa:
Ảo hóa (Virtualization) là kỹ thuật tạo ra phiên bản ảo (virtual) của các tài nguyên máy tính (hardware, operating system, storage, network) thông qua một lớp trừu tượng (abstraction layer), cho phép nhiều môi trường thực thi độc lập chạy đồng thời trên cùng một phần cứng vật lý.
Khái niệm cốt lõi:
┌────────────────────────────────────────────────┐
│     Physical Hardware (Phần cứng vật lý)       │
│     - CPU, RAM, Disk, Network                  │
└──────────────────┬─────────────────────────────┘
                   │
       ┌───────────▼────────────┐
       │  Virtualization Layer  │  ← Abstraction
       │  (Hypervisor/Runtime)  │
       └───────────┬────────────┘
                   │
      ┌────────────┼────────────┐
      │            │            │
┌─────▼─────┐ ┌───▼────┐ ┌────▼─────┐
│ Virtual   │ │Virtual │ │ Virtual  │
│ Machine 1 │ │ Mach 2 │ │ Machine 3│
│           │ │        │ │          │
│ OS + Apps │ │OS+Apps │ │ OS + Apps│
└───────────┘ └────────┘ └──────────┘

Đặc điểm:
- Mỗi VM nghĩ nó chạy trên hardware riêng
- Isolation: VMs không biết về nhau
- Multiple OS có thể chạy đồng thời
Ví dụ cụ thể:
Physical server:
├─ 32 CPU cores
├─ 256 GB RAM
├─ 2 TB SSD

WITHOUT Virtualization:
└─ 1 OS, 1 application
└─ Wasted capacity: 70-80% ❌

WITH Virtualization:
├─ VM1: 8 cores, 64GB RAM → Web server
├─ VM2: 8 cores, 64GB RAM → Database
├─ VM3: 4 cores, 32GB RAM → Cache
├─ VM4: 4 cores, 32GB RAM → Monitoring
└─ Resource utilization: 90%+ ✅

B. Vai trò của Ảo hóa trong Hệ thống Phân tán
1. Resource Consolidation (Hợp nhất tài nguyên)
Traditional Infrastructure:
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Server 1│ │Server 2│ │Server 3│ │Server 4│
│Web     │ │DB      │ │Cache   │ │Backup  │
│(20%    │ │(30%    │ │(15%    │ │(10%    │
│ usage) │ │ usage) │ │ usage) │ │ usage) │
└────────┘ └────────┘ └────────┘ └────────┘
4 servers, average utilization: 18.75% ❌
Cost: 4 × $5,000/year = $20,000

Virtualized Infrastructure:
┌─────────────────────────────────────────┐
│       Single Physical Server            │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐     │
│  │VM1  │ │VM2  │ │VM3  │ │ VM4  │     │
│  │Web  │ │ DB  │ │Cache│ │Backup│     │
│  └─────┘ └─────┘ └─────┘ └──────┘     │
│  Utilization: 75%                       │
└─────────────────────────────────────────┘
1 server, Cost: $8,000/year
Savings: 60% ✅
2. Isolation and Security (Cô lập và Bảo mật)
Security benefits:

VM1 compromised:
├─ Attacker gains root in VM1
├─ Cannot access VM2, VM3, VM4 ✅
├─ Cannot access host OS ✅
└─ Damage contained to VM1 only

Example:
┌──────────────────────────────────┐
│  VM1 (DMZ - Public facing)       │
│  - Web server                    │
│  - Exposed to internet           │
│  - If hacked → Isolated ✅       │
└──────────────────────────────────┘
           │ Firewall rules
┌──────────▼───────────────────────┐
│  VM2 (Internal - Private)        │
│  - Database                      │
│  - Not directly accessible       │
│  - Protected ✅                  │
└──────────────────────────────────┘

Without virtualization:
└─ One OS → Complete compromise ❌
3. Flexibility and Agility (Linh hoạt và Nhanh chóng)
VM lifecycle management:

Create new VM:
├─ Traditional: Days (order hardware, install OS)
├─ Virtual: Minutes (clone template, boot) ✅
└─ 1000x faster provisioning

Move VM:
├─ Physical: Impossible (tied to hardware)
├─ Virtual: Live migration (no downtime) ✅
└─ Example: VMware vMotion

Clone VM:
├─ Physical: Re-install everything
├─ Virtual: Copy file, adjust config ✅
└─ Dev/test environments in seconds

Snapshot:
├─ Physical: Full backup (hours)
├─ Virtual: Instant snapshot ✅
└─ Rollback in case of failure
4. High Availability and Disaster Recovery (Tính sẵn sàng cao)
HA scenario:

Physical Server 1 fails:
┌─────────────────────────────────┐
│  VM1, VM2, VM3 running          │
│  ↓ Server failure detected      │
│  ↓ Live migration triggered     │
│  ↓ VMs moved to Server 2        │
└─────────────────────────────────┘
           ↓
┌─────────────────────────────────┐
│  Physical Server 2              │
│  VM1, VM2, VM3 running ✅       │
│  Downtime: <1 second            │
└─────────────────────────────────┘

Without virtualization:
└─ Server fails → Service down ❌
└─ Manual recovery: Hours
5. Cloud Computing Enabler (Nền tảng điện toán đám mây)
Cloud services built on virtualization:

AWS EC2:
└─ VM instances on shared hardware

Google Compute Engine:
└─ VMs with custom machine types

Azure Virtual Machines:
└─ Windows/Linux VMs on-demand

Pay-per-use model:
├─ Spin up VM: $0.10/hour
├─ Scale to 1000 VMs in minutes
└─ Only possible with virtualization ✅
6. Development and Testing (Phát triển và Kiểm thử)
Dev/Test workflow:

Developer needs:
├─ Linux VM for backend
├─ Windows VM for testing
├─ MacOS VM for iOS build
└─ All on same laptop ✅

Automated testing:
1. Create clean VM snapshot
2. Deploy code
3. Run tests
4. Destroy VM
5. Repeat for next test
→ Isolated, reproducible ✅

CI/CD pipeline:
Jenkins → Spin up VMs → Build → Test → Destroy
→ Parallel testing at scale ✅
7. Server Consolidation in Data Centers
Data center transformation:

Before virtualization (2005):
├─ 10,000 physical servers
├─ Utilization: 15%
├─ Power: 5 MW
├─ Cooling: Massive
└─ Cost: $50M/year

After virtualization (2015):
├─ 1,000 physical servers
├─ 50,000 VMs (50:1 ratio)
├─ Utilization: 70%
├─ Power: 1.5 MW ✅
├─ Cooling: Reduced
└─ Cost: $15M/year (70% savings ✅)

Phần 2: Phân loại Ảo hóa
A. Tổng quan phân loại
Virtualization Classification:

Based on WHAT is virtualized:
├─ Process-level (Instruction Set Architecture)
│  └─ Virtualizes: CPU instructions + Runtime
│  └─ Examples: JVM, .NET CLR, WASM
│
└─ System-level (Full System)
   └─ Virtualizes: Entire computer (CPU, Memory, I/O)
   └─ Examples: VMware, KVM, Xen, Hyper-V

Key difference:
- Process-level: Runs applications
- System-level: Runs operating systems
Visualization:
PROCESS-LEVEL VM:
┌─────────────────────────────────────┐
│      Host Operating System          │
│  ┌───────────────────────────────┐  │
│  │  Runtime Environment (JVM)    │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐     │  │
│  │  │App1 │ │App2 │ │App3 │     │  │
│  │  │.java│ │.java│ │.java│     │  │
│  │  └─────┘ └─────┘ └─────┘     │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
       Hardware


SYSTEM-LEVEL VM:
┌─────────────────────────────────────┐
│         Hypervisor (VMM)            │
│  ┌─────────┐ ┌─────────┐ ┌────────┐│
│  │  VM1    │ │  VM2    │ │  VM3   ││
│  │  ┌───┐  │ │  ┌───┐  │ │ ┌───┐ ││
│  │  │OS1│  │ │  │OS2│  │ │ │OS3│ ││
│  │  │App│  │ │  │App│  │ │ │App│ ││
│  │  └───┘  │ │  └───┘  │ │ └───┘ ││
│  └─────────┘ └─────────┘ └────────┘│
└─────────────────────────────────────┘
       Hardware

Phần 3: Process-Level VM (Máy ảo tiến trình)
A. Định nghĩa và Kiến trúc
Định nghĩa:
Process-level VM (còn gọi là Application VM hay Runtime Environment) ảo hóa tại mức instruction set architecture, cung cấp một môi trường thực thi trừu tượng cho các ứng dụng, độc lập với phần cứng và hệ điều hành cơ sở.
Kiến trúc chi tiết:
┌──────────────────────────────────────────────┐
│        APPLICATION LAYER                     │
│  ┌────────────────────────────────────────┐ │
│  │  Java Application (.jar)               │ │
│  │  - Source: .java files                 │ │
│  │  - Compiled to: .class (bytecode)      │ │
│  └──────────────┬─────────────────────────┘ │
│                 │                            │
│  ┌──────────────▼─────────────────────────┐ │
│  │  JAVA VIRTUAL MACHINE (JVM)           │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐ │ │
│  │  │  Class Loader                    │ │ │
│  │  │  - Load .class files             │ │ │
│  │  │  - Verify bytecode               │ │ │
│  │  └──────────────────────────────────┘ │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐ │ │
│  │  │  Bytecode Verifier               │ │ │
│  │  │  - Security checks               │ │ │
│  │  │  - Type safety                   │ │ │
│  │  └──────────────────────────────────┘ │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐ │ │
│  │  │  Execution Engine                │ │ │
│  │  │  ┌─────────────┐ ┌─────────────┐│ │ │
│  │  │  │ Interpreter │ │ JIT Compiler││ │ │
│  │  │  └─────────────┘ └─────────────┘│ │ │
│  │  └──────────────────────────────────┘ │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐ │ │
│  │  │  Runtime Data Areas              │ │ │
│  │  │  - Heap (objects)                │ │ │
│  │  │  - Method area (class metadata)  │ │ │
│  │  │  - Stack (per thread)            │ │ │
│  │  │  - PC registers                  │ │ │
│  │  └──────────────────────────────────┘ │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐ │ │
│  │  │  Garbage Collector               │ │ │
│  │  │  - Automatic memory management   │ │ │
│  │  └──────────────────────────────────┘ │ │
│  └────────────────────────────────────────┘ │
│                 │                            │
│  ┌──────────────▼─────────────────────────┐ │
│  │  Native Method Interface (JNI)        │ │
│  └────────────────────────────────────────┘ │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│      HOST OPERATING SYSTEM                   │
│      (Windows, Linux, macOS)                 │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│      PHYSICAL HARDWARE                       │
│      (CPU, Memory, Disk)                     │
└──────────────────────────────────────────────┘

B. Cơ chế hoạt động (JVM Example)
1. Compilation và Loading:
java// Step 1: Source code (.java)
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// Step 2: Compile to bytecode (.class)
$ javac HelloWorld.java

// Generated bytecode (simplified):
public class HelloWorld {
  public static void main(String[] args);
    Code:
       0: getstatic     #2  // Field System.out
       3: ldc           #3  // String "Hello, World!"
       5: invokevirtual #4  // Method println
       8: return
}

// Step 3: JVM loads and executes
$ java HelloWorld

JVM Process:
1. Class Loader reads HelloWorld.class
2. Verifier checks bytecode validity
3. Execution Engine runs bytecode
```

**2. Bytecode Execution - Two Modes:**
```
MODE 1: INTERPRETATION (Slow but simple)

Bytecode:  getstatic #2
           ↓
Interpreter: 
  read instruction → decode → execute
  (One instruction at a time)

Performance: Slow ❌
- Overhead per instruction
- No optimization

Example timeline:
Instruction 1: 10 cycles
Instruction 2: 10 cycles
Instruction 3: 10 cycles
Total: 30 cycles
```
```
MODE 2: JIT COMPILATION (Fast but complex)

Bytecode:  getstatic #2
           ldc #3
           invokevirtual #4
           ↓
JIT Compiler:
  Detect hot code (frequently executed)
  → Compile to native machine code
  → Cache compiled code
  → Execute native code directly

Performance: Fast ✅
- One-time compilation cost
- Native speed for hot paths

Example timeline:
Compilation: 1000 cycles (one-time)
Instruction 1-100: 2 cycles each (native)
Total: 1000 + 200 = 1200 cycles

After warm-up:
Instruction 101-1000: 2 cycles each
Total: 1800 cycles (vs 9000 interpreted!)
JIT Compilation example:
java// Hot loop detected
for (int i = 0; i < 1_000_000; i++) {
    sum += array[i];  // ← JIT compiles this
}

Bytecode:
  iload_1          // load i
  aload_0          // load array
  iaload           // array[i]
  iadd             // sum += 
  istore_2         // store sum
  iinc 1, 1        // i++

JIT compiles to (x86 assembly):
  mov eax, [ebp-4]    // load i
  mov ebx, [ebp-8]    // load array base
  add eax, [ebx+ecx]  // sum += array[i]
  inc ecx             // i++
  
→ 10x faster than interpretation ✅
```

---

**3. Memory Management (Garbage Collection):**
```
JVM Heap Layout:

┌────────────────────────────────────────┐
│           JAVA HEAP                    │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │  Young Generation                │ │
│  │  ┌────────┐ ┌──────────────────┐│ │
│  │  │ Eden   │ │ Survivor Spaces  ││ │
│  │  │        │ │  S0  │  S1       ││ │
│  │  └────────┘ └──────────────────┘│ │
│  │  New objects allocated here      │ │
│  └──────────────────────────────────┘ │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │  Old Generation (Tenured)        │ │
│  │  Long-lived objects              │ │
│  └──────────────────────────────────┘ │
└────────────────────────────────────────┘

GC Process:
1. Minor GC (Young Generation):
   - Frequent (every few seconds)
   - Fast (<10ms)
   - Collects short-lived objects
   
2. Major GC (Old Generation):
   - Infrequent (every few minutes)
   - Slow (100ms-1s)
   - Collects long-lived objects

Example:
Object obj = new Object();  // Allocated in Eden
// ... used for a while ...
// obj becomes unreachable
// → Minor GC collects it ✅

// No manual free() needed! ✅
```

---

**C. Ưu điểm của Process-Level VM**

**1. Platform Independence (Độc lập nền tảng)**
```
Write Once, Run Anywhere (WORA):

Java code:
┌─────────────────────────────────┐
│  HelloWorld.java                │
│  public class HelloWorld {      │
│    // ... code ...               │
│  }                              │
└─────────────────────────────────┘
          ↓ javac (compile once)
┌─────────────────────────────────┐
│  HelloWorld.class (bytecode)    │
└─────────────────────────────────┘
          ↓ Run on any platform
    ┌─────┴─────┬─────────┬──────┐
    ↓           ↓         ↓      ↓
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Windows │ │ Linux  │ │ macOS  │ │Android │
│  JVM   │ │  JVM   │ │  JVM   │ │  JVM   │
└────────┘ └────────┘ └────────┘ └────────┘

Same bytecode runs everywhere! ✅

Contrast with C/C++:
┌─────────────┐
│ program.c   │
└──────┬──────┘
       ├─── gcc (Windows) → program.exe
       ├─── gcc (Linux)   → program (ELF)
       └─── gcc (macOS)   → program (Mach-O)

Must recompile for each platform! ❌
```

**2. Security (Bảo mật)**
```
Bytecode Verification:

JVM Security checks:
┌─────────────────────────────────────┐
│ 1. Type Safety                      │
│    - No illegal type casts          │
│    - Array bounds checking          │
│                                     │
│ 2. No Invalid Pointers              │
│    - No pointer arithmetic          │
│    - No manual memory access        │
│                                     │
│ 3. Access Control                   │
│    - private/protected/public       │
│    - Cannot access private fields   │
│                                     │
│ 4. Sandbox                          │
│    - Untrusted code isolated        │
│    - Limited file/network access    │
└─────────────────────────────────────┘

Example: Malicious bytecode rejected
public void hack() {
    // Try to access memory directly
    // JVM verifier: REJECTED ❌
}

vs Native code:
void hack() {
    int* ptr = (int*)0xDEADBEEF;
    *ptr = 0;  // Allowed! Crash system! ❌
}
3. Automatic Memory Management
java// Java (with GC)
public void createObjects() {
    for (int i = 0; i < 1000000; i++) {
        Object obj = new Object();  // Allocate
        // Use obj
        // No explicit free needed! ✅
    }
    // GC automatically collects unreachable objects
}

// C (manual memory)
void createObjects() {
    for (int i = 0; i < 1000000; i++) {
        Object* obj = malloc(sizeof(Object));
        // Use obj
        free(obj);  // MUST free manually ❌
        // Forget free() → Memory leak ❌
    }
}

Benefits:
✅ No memory leaks (if no circular refs)
✅ No dangling pointers
✅ No double-free bugs
❌ GC pauses (trade-off)
```

**4. Dynamic Optimization**
```
JIT Profiling and Optimization:

// Java code
int sum(int[] array) {
    int result = 0;
    for (int i = 0; i < array.length; i++) {
        result += array[i];
    }
    return result;
}

JVM Runtime:
1. Initial: Interpret bytecode (slow)
2. After 10,000 calls: JIT compiles
3. JIT optimizations:
   - Loop unrolling
   - Bounds check elimination
   - SIMD instructions
   - Inlining

Optimized native code:
// Vectorized loop (SIMD)
__m128i sum = _mm_setzero_si128();
for (int i = 0; i < n; i += 4) {
    __m128i data = _mm_loadu_si128(&array[i]);
    sum = _mm_add_epi32(sum, data);
}

Result: Near-native performance ✅
Sometimes faster than C! (adaptive optimization)
```

---

**D. Nhược điểm của Process-Level VM**

**1. Startup Overhead**
```
Application startup time:

Native C application:
├─ Load executable: 10ms
├─ Initialize: 5ms
└─ Total: 15ms ✅

Java application:
├─ Start JVM: 500ms ❌
├─ Load classes: 200ms
├─ Initialize JVM: 100ms
├─ JIT warm-up: 300ms
└─ Total: 1,100ms ❌

70x slower startup! ❌

Impact:
- Command-line tools: Poor user experience
- Serverless (Lambda): Cold start problem
- Desktop apps: Slow initial launch
```

**2. Memory Footprint**
```
Memory usage comparison:

"Hello World" program:

C:
├─ Executable: 16 KB
├─ Runtime memory: 1 MB
└─ Total: ~1 MB ✅

Java:
├─ .class file: 416 bytes
├─ JVM: 50 MB (minimum heap)
├─ Runtime: 20 MB (actual usage)
└─ Total: 70 MB ❌

70x more memory! ❌

For microservices:
1000 instances × 70 MB = 70 GB
vs
1000 instances × 1 MB = 1 GB

Cost impact: Significant ❌
```

**3. GC Pauses**
```
Garbage Collection impact:

Application timeline:
Time →
├─ Running: 100ms
├─ GC pause: 50ms ❌ (Stop-The-World)
├─ Running: 100ms
├─ GC pause: 50ms ❌
└─ ...

Throughput: 66% (100 / 150)
Latency spikes: 50ms every 150ms

Impact on real-time systems:
- Trading systems: Unacceptable ❌
- Gaming: Stuttering ❌
- Audio/video: Glitches ❌

Mitigations:
- G1GC: Lower pauses (~10ms)
- ZGC: <1ms pauses ✅
- Shenandoah: Concurrent GC
```

**4. Performance Ceiling**
```
Peak performance comparison:

Highly optimized C:
└─ 100% native performance (baseline) ✅

Java (JIT compiled):
├─ Warm: 85-95% of C
├─ Cold: 10-20% of C (interpreted)
└─ Average: 80-90% ⚠️

Reasons for gap:
- GC overhead: ~5-10%
- Bounds checking: ~2-5%
- Virtual dispatch: ~1-3%
- Object overhead: ~2-5%

For CPU-intensive tasks:
- C: 1000 operations/sec
- Java: 850 operations/sec
- Gap matters at scale ⚠️
```

---

**E. Ví dụ khác: .NET CLR**
```
.NET Common Language Runtime (CLR):

Similar to JVM:
┌─────────────────────────────────┐
│  C#/F#/VB.NET Source Code       │
└────────────┬────────────────────┘
             ↓ Compile
┌─────────────────────────────────┐
│  Common Intermediate Language   │
│  (CIL / MSIL bytecode)          │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  .NET Runtime (CLR)             │
│  - JIT Compiler (RyuJIT)        │
│  - Garbage Collector            │
│  - Type System                  │
│  - Security                     │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  Native Machine Code            │
└─────────────────────────────────┘

Differences from JVM:
✅ Better Windows integration
✅ Multiple languages (C#, F#, VB)
✅ Ahead-of-Time compilation option (Native AOT)
⚠️ Historically Windows-only (now cross-platform)
```

---

**F. WebAssembly (WASM) - Modern Example**
```
WebAssembly Architecture:

┌─────────────────────────────────┐
│  C/C++/Rust Source              │
└────────────┬────────────────────┘
             ↓ Compile
┌─────────────────────────────────┐
│  WebAssembly Bytecode (.wasm)   │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  Browser / WASM Runtime         │
│  - JIT / AOT compilation        │
│  - Sandboxed execution          │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  Native Machine Code            │
└─────────────────────────────────┘

Benefits:
✅ Near-native performance (95%+)
✅ Language agnostic (C/C++/Rust/etc.)
✅ Browser integration
✅ Smaller than JS (binary format)

Use cases:
- Web games (Unity, Unreal)
- Video/audio processing
- CAD software in browser
- Cryptography
```

---

#### **Phần 4: System-Level VM (Hypervisor-based)**

**A. Định nghĩa và Kiến trúc**

**Định nghĩa:**
System-level VM (hay Hypervisor-based VM) ảo hóa toàn bộ máy tính vật lý, bao gồm CPU, memory, storage, và I/O devices, cho phép chạy nhiều hệ điều hành độc lập (guest OS) trên cùng một phần cứng vật lý.

**Kiến trúc Type 1 Hypervisor (Bare-metal):**
```
┌─────────────────────────────────────────────┐
│              HARDWARE                       │
│  CPU (Intel VT-x / AMD-V)                  │
│  Memory (Physical RAM)                      │
│  Storage (Disk controllers)                │
│  Network (NICs)                            │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         HYPERVISOR (Type 1)                 │
│         (VMware ESXi, Xen, KVM)            │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  Virtual Machine Monitor (VMM)      │   │
│  │  - CPU virtualization               │   │
│  │  - Memory management                │   │
│  │  - I/O virtualization               │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  Scheduler & Resource Manager       │   │
│  └─────────────────────────────────────┘   │
└──────────────────┬──────────────────────────┘
                   │
      ┌────────────┼────────────┐
      │            │            │
┌─────▼─────┐ ┌───▼────┐ ┌────▼─────┐
│   VM 1    │ │  VM 2  │ │   VM 3   │
│           │ │        │ │          │
│ ┌───────┐ │ │┌──────┐│ │┌────────┐│
│ │Linux  │ │ ││Windows││ ││ BSD    ││
│ │Kernel │ │ ││Kernel ││ ││ Kernel ││
│ └───┬───┘ │ │└───┬──┘│ │└───┬────┘│
│     │     │ │    │   │ │    │     │
│ ┌───▼───┐ │ │┌───▼──┐│ │┌───▼────┐│
│ │ Apps  │ │ ││ Apps ││ ││  Apps  ││
│ └───────┘ │ │└──────┘│ │└────────┘│
└───────────┘ └────────┘ └──────────┘

Virtual Hardware Interface:
Each VM sees:
- Virtual CPU (vCPU)
- Virtual Memory
- Virtual Disk (VMDK/VHD)
- Virtual NIC
Phần 4: System-Level VM (tiếp theo)
B. Cơ chế hoạt động của Hypervisor
1. CPU Virtualization (Ảo hóa CPU)
Hardware-assisted Virtualization:
Intel VT-x / AMD-V Architecture:

┌─────────────────────────────────────────────┐
│         PHYSICAL CPU                        │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  VMX Root Mode (Host mode)            │ │
│  │  - Hypervisor runs here               │ │
│  │  - Full hardware access               │ │
│  └───────────────┬───────────────────────┘ │
│                  │ VM Entry/Exit           │
│  ┌───────────────▼───────────────────────┐ │
│  │  VMX Non-root Mode (Guest mode)       │ │
│  │  - Guest OS runs here                 │ │
│  │  - Restricted hardware access         │ │
│  └───────────────────────────────────────┘ │
└─────────────────────────────────────────────┘

Privileged instructions:
Guest OS executes: CLI (disable interrupts)
         ↓
CPU traps to hypervisor (VM Exit)
         ↓
Hypervisor emulates instruction
         ↓
Return to guest (VM Entry)
Virtual CPU (vCPU) scheduling:
Physical: 8-core CPU
VMs: 3 VMs with 4 vCPUs each

┌────────────────────────────────────────┐
│  Physical CPU Cores (8 cores)         │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │
│  │Core 0│ │Core 1│ │Core 2│ │Core 3│ │
│  └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘ │
│      │        │        │        │     │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │
│  │Core 4│ │Core 5│ │Core 6│ │Core 7│ │
│  └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘ │
└──────┼────────┼────────┼────────┼─────┘
       │        │        │        │
┌──────▼────────▼────────▼────────▼─────┐
│      Hypervisor Scheduler              │
│  Time-slice vCPUs across pCPUs         │
└──────┬────────┬────────┬────────┬─────┘
       │        │        │        │
   ┌───▼───┐ ┌─▼───┐ ┌──▼───┐
   │ VM1   │ │VM2  │ │ VM3  │
   │4 vCPUs│ │4vCPU│ │4vCPUs│
   └───────┘ └─────┘ └──────┘

Overcommit ratio: 12 vCPUs / 8 pCPUs = 1.5:1 ✅
(Common: 2:1 to 4:1 for typical workloads)

Scheduling timeline:
Time →
Core 0: [VM1.vCPU0] [VM2.vCPU0] [VM3.vCPU0] [VM1.vCPU0]
Core 1: [VM1.vCPU1] [VM2.vCPU1] [VM3.vCPU1] [VM1.vCPU1]
...
CPU overhead:
Instruction execution:

Native (no virtualization):
Instruction → CPU executes directly
Time: 1-5 cycles ✅

With hardware virtualization (VT-x):
Normal instruction → CPU executes directly
Time: 1-5 cycles ✅ (same!)

Privileged instruction (e.g., CLI):
Instruction → VM Exit → Hypervisor → Emulate → VM Entry
Time: 100-500 cycles ⚠️

Overhead: ~1-5% for typical workloads ✅
(Most instructions are not privileged)

2. Memory Virtualization (Ảo hóa bộ nhớ)
Two-level page tables:
Memory address translation:

Guest Virtual Address (GVA)
         ↓
Guest Page Table (maintained by Guest OS)
         ↓
Guest Physical Address (GPA)
         ↓
Extended Page Table (EPT) / Nested Page Table (NPT)
(maintained by Hypervisor)
         ↓
Host Physical Address (HPA)

Example:
┌─────────────────────────────────────┐
│  VM1: "0x1000" (Guest Virtual)      │
└──────────────┬──────────────────────┘
               ↓ Guest PT
┌─────────────────────────────────────┐
│  VM1: "0x50000" (Guest Physical)    │
└──────────────┬──────────────────────┘
               ↓ EPT (Hypervisor)
┌─────────────────────────────────────┐
│  Host: "0xA3450000" (Host Physical) │
└─────────────────────────────────────┘

Hardware support (Intel EPT / AMD NPT):
- CPU walks both tables automatically
- Minimal overhead (~3-5%) ✅
Memory overcommit:
Physical RAM: 256 GB

VM1: 100 GB allocated
VM2: 100 GB allocated
VM3: 100 GB allocated
Total: 300 GB (overcommitted!)

How it works:
┌─────────────────────────────────────┐
│  Hypervisor Memory Management       │
│                                     │
│  1. Transparent Page Sharing (TPS)  │
│     - Deduplicate identical pages   │
│     - Example: 3 VMs running same OS│
│     - Same kernel pages shared      │
│     - Saves: ~30-40 GB ✅          │
│                                     │
│  2. Ballooning                      │
│     - Guest cooperates with host    │
│     - Balloon driver inflates       │
│     - Guest releases unused memory  │
│     - Saves: ~20-30 GB ✅          │
│                                     │
│  3. Memory Compression              │
│     - Compress inactive pages       │
│     - 2:1 compression ratio         │
│     - Saves: ~15-20 GB ✅          │
│                                     │
│  4. Swapping (last resort)          │
│     - Swap to disk                  │
│     - Slow but prevents OOM ⚠️      │
└─────────────────────────────────────┘

Result:
300 GB virtual → ~200 GB physical ✅
Example: Memory ballooning:
c// Balloon driver in Guest OS
void inflate_balloon(int target_pages) {
    for (int i = 0; i < target_pages; i++) {
        void* page = alloc_page();  // Allocate in guest
        
        // Tell hypervisor we're using this page
        hypercall(BALLOON_INFLATE, page);
        
        // Hypervisor reclaims underlying physical page
        // for use by other VMs
    }
}

Timeline:
1. Hypervisor detects memory pressure
2. Sends balloon target to VM1: "Inflate to 1GB"
3. Balloon driver allocates 1GB in guest
4. Guest OS thinks it's low on memory
5. Guest frees caches, releases memory
6. Hypervisor reclaims 1GB physical RAM
7. Gives RAM to VM2 (memory-hungry) ✅
```

---

**3. I/O Virtualization (Ảo hóa I/O)**

**Three approaches:**

**A. Full Device Emulation (Slow)**
```
┌─────────────────────────────────────┐
│  Guest OS                           │
│  ┌───────────────────────────────┐  │
│  │  Device Driver (e1000)        │  │
│  │  (thinks it's Intel NIC)      │  │
│  └────────────┬──────────────────┘  │
│               │ I/O operations      │
└───────────────┼─────────────────────┘
                │ VM Exit
┌───────────────▼─────────────────────┐
│  Hypervisor                         │
│  ┌───────────────────────────────┐  │
│  │  Device Emulation             │  │
│  │  - Emulates e1000 registers   │  │
│  │  - Translates I/O operations  │  │
│  └────────────┬──────────────────┘  │
└───────────────┼─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│  Physical NIC (Broadcom)            │
└─────────────────────────────────────┘

Performance:
- Every I/O = VM Exit (expensive!)
- Software emulation overhead
- Throughput: ~1 Gbps (poor ❌)
```

**B. Para-virtualization (Fast)**
```
┌─────────────────────────────────────┐
│  Guest OS (modified)                │
│  ┌───────────────────────────────┐  │
│  │  Virtio Driver                │  │
│  │  (knows it's virtualized!)    │  │
│  └────────────┬──────────────────┘  │
│               │ Hypercalls          │
└───────────────┼─────────────────────┘
                │ Direct interface
┌───────────────▼─────────────────────┐
│  Hypervisor                         │
│  ┌───────────────────────────────┐  │
│  │  Virtio Backend               │  │
│  │  - Shared memory rings        │  │
│  │  - Minimal overhead           │  │
│  └────────────┬──────────────────┘  │
└───────────────┼─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│  Physical NIC                       │
└─────────────────────────────────────┘

Performance:
- Fewer VM Exits (batching)
- Direct memory access
- Throughput: ~8 Gbps (good ✅)

Code example:
// Virtio ring buffer (shared)
struct virtio_ring {
    descriptor_t descriptors[QUEUE_SIZE];
    uint16_t avail_idx;  // Guest writes here
    uint16_t used_idx;   // Hypervisor writes here
};

// Guest sends packet
virtio_ring->descriptors[idx] = packet;
virtio_ring->avail_idx++;
kick_hypervisor();  // Notify (1 VM exit)

// Hypervisor processes (batch of packets)
while (avail_idx > used_idx) {
    process_packet(descriptors[used_idx]);
    used_idx++;
}
```

**C. SR-IOV (Fastest - Hardware passthrough)**
```
┌─────────────────────────────────────┐
│  Guest OS                           │
│  ┌───────────────────────────────┐  │
│  │  Native NIC Driver            │  │
│  └────────────┬──────────────────┘  │
└───────────────┼─────────────────────┘
                │ DIRECT ACCESS
┌───────────────▼─────────────────────┐
│  SR-IOV NIC                         │
│  ┌─────────┐ ┌─────────┐ ┌───────┐ │
│  │  VF 1   │ │  VF 2   │ │  VF 3 │ │
│  │(for VM1)│ │(for VM2)│ │(VM3)  │ │
│  └─────────┘ └─────────┘ └───────┘ │
│         Physical Function            │
└─────────────────────────────────────┘

Performance:
- Zero VM Exits for I/O! ✅
- Native performance
- Throughput: 10+ Gbps ✅✅

Trade-off:
- Less flexible (VM tied to physical NIC)
- Cannot live migrate easily ⚠️
- Requires hardware support ⚠️
```

**Network I/O benchmark:**
```
Throughput test (10 Gbps NIC):

Full emulation (e1000):
├─ Throughput: 1.2 Gbps
├─ CPU overhead: 80%
└─ Latency: 500 μs
Rating: 1/10 ❌

Para-virtualization (virtio):
├─ Throughput: 8.5 Gbps
├─ CPU overhead: 20%
└─ Latency: 50 μs
Rating: 8/10 ✅

SR-IOV passthrough:
├─ Throughput: 9.8 Gbps
├─ CPU overhead: 5%
└─ Latency: 10 μs
Rating: 10/10 ✅✅
```

---

**4. Storage Virtualization**
```
Virtual Disk formats:

VMDK (VMware):
┌─────────────────────────────────────┐
│  VM sees: 100 GB disk               │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  VMDK file (thin provisioned)       │
│  - Actually uses: 30 GB             │
│  - Grows on demand                  │
│  - Max: 100 GB                      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Host filesystem (VMFS/NFS)         │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Physical disk                      │
└─────────────────────────────────────┘

Benefits:
✅ Space efficient (thin provisioning)
✅ Snapshots (point-in-time copy)
✅ Easy backup (copy file)
✅ Live migration (copy + sync)

Overhead: ~5-15% ⚠️
```

**Storage I/O path:**
```
Guest write operation:

1. Guest OS: write(fd, buffer, size)
   ↓
2. Guest filesystem: ext4/NTFS
   ↓
3. Virtual disk driver (SCSI/SATA)
   ↓ VM Exit
4. Hypervisor: VMDK/VHD handler
   - Translate guest LBA → host offset
   - Check thin provisioning
   - Allocate new blocks if needed
   ↓
5. Host filesystem: XFS/VMFS
   ↓
6. Physical disk controller
   ↓
7. Physical disk: Write to platter/SSD

Layers: 7 (vs 4 for native) ⚠️
Overhead: 10-20% latency increase
```

---

**C. Ưu điểm của System-Level VM**

**1. Strong Isolation (Cô lập mạnh mẽ)**
```
VM isolation:

┌─────────────────────────────────────┐
│  VM1 (Web Server)                   │
│  - Public facing                    │
│  - Compromised by attacker 🔴       │
│                                     │
│  Attacker gains root in VM1         │
│  ├─ Cannot read VM2 memory ✅      │
│  ├─ Cannot access VM2 files ✅     │
│  ├─ Cannot see VM2 processes ✅    │
│  └─ Hardware-enforced isolation ✅ │
└─────────────────────────────────────┘
               ║ Firewall
┌─────────────────────────────────────┐
│  VM2 (Database)                     │
│  - Private                          │
│  - SAFE ✅                          │
└─────────────────────────────────────┘

vs Containers:
Container 1 compromised → Kernel shared
→ Kernel exploit → ALL containers compromised ❌
```

**2. Full OS Flexibility**
```
Multiple OS on same hardware:

Physical Server:
├─ VM1: Windows Server 2022
│  └─ .NET applications
│
├─ VM2: Ubuntu 22.04 LTS
│  └─ Python/Django apps
│
├─ VM3: CentOS 7
│  └─ Legacy Java apps
│
├─ VM4: FreeBSD 13
│  └─ Network appliance
│
└─ VM5: Custom Linux kernel
   └─ Embedded systems testing

Impossible without virtualization! ✅
```

**3. Live Migration (Di chuyển trực tiếp)**
```
VM live migration (vMotion/Live Migration):

Timeline:
t=0s:  VM running on Host A
       ├─ Start migration
       ├─ Copy memory pages to Host B
       │  (VM still running on A)
       │
t=10s: ├─ Most memory copied
       ├─ Track dirty pages
       │
t=15s: ├─ Pause VM briefly (~100ms)
       ├─ Copy final dirty pages
       ├─ Transfer CPU state
       │
t=15.1s: VM running on Host B ✅
       └─ Total downtime: 100ms (imperceptible!)

Use cases:
✅ Hardware maintenance (no downtime)
✅ Load balancing (move VMs to less loaded host)
✅ Power optimization (consolidate, power off hosts)

Code example (simplified):
1. while (dirty_pages > threshold) {
       copy_pages(src_host, dst_host);
       track_dirty_pages();
   }
2. pause_vm();
3. copy_final_state();
4. resume_vm_on_dst();
```

**4. Snapshot and Rollback**
```
VM snapshots:

Original VM state:
┌─────────────────────────────────────┐
│  VM Disk: 50 GB                     │
│  Memory: 16 GB                      │
│  CPU state: Registers, PC           │
└─────────────────────────────────────┘
         ↓ Snapshot (instant!)
┌─────────────────────────────────────┐
│  Snapshot 1 (T0)                    │
│  - Disk: COW pointer (0 bytes)      │
│  - Memory: Saved (16 GB)            │
│  - CPU: Saved (few KB)              │
└─────────────────────────────────────┘

Continue running:
VM makes changes → New blocks written
         ↓ Snapshot
┌─────────────────────────────────────┐
│  Snapshot 2 (T1)                    │
│  - Delta: 5 GB                      │
└─────────────────────────────────────┘

Rollback to Snapshot 1:
├─ Discard all changes after T0
├─ Restore memory state
└─ Resume from saved CPU state
Time: <1 second ✅

Use cases:
✅ Before software updates
✅ Testing/development
✅ Malware analysis
✅ Training environments
```

**5. Resource Allocation and QoS**
```
Resource controls:

VM1 (Production):
├─ CPU: 8 vCPUs, shares=2000 (high)
├─ Memory: 32 GB, reservation=24 GB
├─ Disk: 10,000 IOPS limit
└─ Network: 5 Gbps guaranteed

VM2 (Development):
├─ CPU: 4 vCPUs, shares=500 (low)
├─ Memory: 8 GB, no reservation
├─ Disk: 1,000 IOPS limit
└─ Network: Best effort

Under contention:
Hypervisor enforces:
├─ VM1 gets 80% CPU (2000/(2000+500))
├─ VM2 gets 20% CPU
├─ VM1 memory never swapped (reserved)
└─ VM2 may be swapped/ballooned ⚠️

Guaranteed QoS ✅
```

---

**D. Nhược điểm của System-Level VM**

**1. Performance Overhead**
```
Overhead breakdown:

CPU:
├─ Native: 100% (baseline)
├─ VM: 95-98% ⚠️
└─ Overhead: 2-5% (VM Exits, scheduling)

Memory:
├─ Native: 100%
├─ VM: 95-97% ⚠️
└─ Overhead: 3-5% (EPT walks, TLB misses)

I/O (virtio):
├─ Native: 100%
├─ VM: 80-90% ⚠️
└─ Overhead: 10-20% (emulation, exits)

Overall: 5-10% performance loss ⚠️

CPU-intensive benchmark:
Native: 1000 operations/sec
VM:     920 operations/sec (8% slower)

I/O-intensive benchmark:
Native: 10 Gbps throughput
VM:     8.5 Gbps (15% slower)
```

**2. Resource Overhead**
```
Memory overhead per VM:

Guest OS: 2 GB
Guest apps: 4 GB
Total guest memory: 6 GB

Hypervisor overhead:
├─ VM control structures: 100 MB
├─ Virtual device emulation: 50 MB
├─ Shadow page tables: 200 MB (if no EPT)
├─ vCPU contexts: 50 MB
└─ Total overhead: ~400 MB per VM

10 VMs = 4 GB overhead ⚠️

Plus hypervisor itself: 500 MB - 2 GB

Total: ~5-6 GB for infrastructure ⚠️
```

**3. Complex Management**
```
VM lifecycle management:

Tasks:
├─ Provisioning: Configure CPU, memory, disk, network
├─ Patching: Update guest OS, applications
├─ Monitoring: Track performance, health
├─ Backup: Snapshot, replicate
├─ Security: Firewall, antivirus, updates
└─ Licensing: Windows/RHEL per VM

Tools needed:
├─ vCenter (VMware): $5,000-$20,000
├─ SCVMM (Hyper-V): Included with Windows
├─ oVirt (KVM): Free but complex
└─ Learning curve: Months ⚠️

Compare to bare metal:
- 1 server = 1 OS to manage
- 1 server with 10 VMs = 10 OS to manage ⚠️
```

**4. Licensing Costs**
```
Cost comparison:

Bare metal server:
├─ Hardware: $10,000
├─ Windows Server: $1,000 (1 license)
└─ Total: $11,000

Virtualized (10 VMs):
├─ Hardware: $10,000
├─ Hypervisor (VMware ESXi): $5,000
├─ vCenter: $5,000
├─ Windows Server: $10,000 (10 licenses)
└─ Total: $30,000 ⚠️

Note: Licensing varies
- VMware: Per socket
- Microsoft: Per core
- RHEL: Per VM
- Free options: KVM, Xen, Proxmox ✅
```

---

#### **Phần 5: So sánh Process-level VM vs System-level VM**

**Bảng so sánh tổng quan:**

| Tiêu chí | Process-Level VM | System-Level VM |
|----------|------------------|-----------------|
| **Scope** | Single application | Entire OS |
| **Isolation** | Process-level | Hardware-level ✅ |
| **Guest OS** | Không cần | Cần (full OS) |
| **Startup time** | 1-3 seconds | 30-120 seconds ⚠️ |
| **Memory overhead** | 50-200 MB | 500 MB - 2 GB ⚠️ |
| **Performance** | 80-95% native | 90-98% native ✅ |
| **Portability** | Code portable ✅ | VM image portable ✅ |
| **Security** | Moderate | Strong ✅ |
| **Management** | Simple ✅ | Complex ⚠️ |

---

**Chi tiết so sánh:**

**1. Architecture Scope**
```
PROCESS-LEVEL VM:

┌──────────────────────────────────────┐
│      Host OS (Windows/Linux)         │
│  ┌────────────────────────────────┐  │
│  │  JVM / CLR                     │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐   │  │
│  │  │App 1 │ │App 2 │ │App 3 │   │  │
│  │  │.jar  │ │.jar  │ │.jar  │   │  │
│  │  └──────┘ └──────┘ └──────┘   │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

Scope: Application only
OS: Shared (single OS)


SYSTEM-LEVEL VM:

┌──────────────────────────────────────┐
│        Hypervisor                    │
│  ┌───────────┐ ┌───────────┐        │
│  │   VM 1    │ │   VM 2    │        │
│  │  ┌─────┐  │ │  ┌─────┐  │        │
│  │  │OS 1 │  │ │  │OS 2 │  │        │
│  │  │Apps │  │ │  │Apps │  │        │
│  │  └─────┘  │ │  └─────┘  │        │
│  └───────────┘ └───────────┘        │
└──────────────────────────────────────┘

Scope: Entire system
OS: Separate (multiple OS)
```

**2. Isolation Strength**
```
PROCESS-LEVEL VM:

Security boundary: Process isolation
┌─────────────────────────────────────┐
│  Operating System                   │
│  ┌─────────┐ ┌─────────┐           │
│  │ JVM 1   │ │ JVM 2   │           │
│  │ App A   │ │ App B   │           │
│  └─────────┘ └─────────┘           │
│       └──────────┘                  │
│     Same kernel                     │
│     Shared kernel exploits affect   │
│     both JVMs ⚠️                    │
└─────────────────────────────────────┘

Isolation: Moderate (process boundaries)


SYSTEM-LEVEL VM:

Security boundary: Hardware-enforced
┌─────────────────────────────────────┐
│  Hypervisor                         │
│  ┌──────────┐ ┌──────────┐         │
│  │ VM1      │ │ VM2      │         │
│  │ OS + App │ │ OS + App │         │
│  └──────────┘ └──────────┘         │
│       ║            ║                │
│   Separate     Separate             │
│   memory       CPU context          │
│                                     │
│   VM1 exploit → VM2 safe ✅        │
└─────────────────────────────────────┘

Isolation: Strong (hardware-level)
```

**3. Resource Overhead**
```
"Hello World" application:

PROCESS-LEVEL (Java):
┌─────────────────────────────────────┐
│  JVM overhead: 50-100 MB            │
│  App code: 1 MB                     │
│  Total: 51-101 MB                   │
│  Startup: 1-2 seconds               │
└─────────────────────────────────────┘

SYSTEM-LEVEL (VM):
┌─────────────────────────────────────┐
│  Guest OS: 512 MB - 2 GB ⚠️         │
│  JVM overhead: 50-100 MB            │
│  App code: 1 MB                     │
│  VM overhead: 100-200 MB            │
│  Total: 650 MB - 2.3 GB ⚠️          │
│  Startup: 30-60 seconds ⚠️          │
└─────────────────────────────────────┘

Overhead ratio: ~20x ⚠️
```

**4. Portability**
```
PROCESS-LEVEL:

Source code portability:
┌──────────────────┐
│  HelloWorld.java │
└────────┬─────────┘
         ↓ Compile once
┌──────────────────┐
│ HelloWorld.class │ ← Portable bytecode
└────────┬─────────┘
    ┌────┴────┬─────────┬────────┐
    ↓         ↓         ↓        ↓
┌────────┐ ┌────┐ ┌─────────┐ ┌─────┐
│Windows │ │Linux│ │  macOS  │ │ ARM │
│  JVM   │ │ JVM │ │   JVM   │ │ JVM │
└────────┘ └────┘ └─────────┘ └─────┘

Write once, run anywhere ✅
Requirement: JVM installed


SYSTEM-LEVEL:

VM image portability:
┌────────────────────────────────┐
│  VM Image (VMDK/VHD)           │
│  - OS: Linux                   │
│  - Apps: Installed             │
│  - Config: Baked in            │
└──────────────┬─────────────────┘
               ↓ Move VM
    ┌──────────┼──────────┬───────────┐
    ↓          ↓          ↓           ↓
┌────────┐ ┌──────┐ ┌────────┐ ┌──────┐
│ ESXi   │ │ KVM  │ │Hyper-V │ │ Xen  │
└────────┘ └──────┘ └────────┘ └──────┘

Move VM, not code ✅
Requirement: Hypervisor
```

**5. Performance Characteristics**
```
CPU-bound workload (Prime number calculation):

Native C:
├─ Time: 10.0 seconds (baseline)
└─ Throughput: 100%

Process-level (Java JIT):
├─ Time: 10.5 seconds
├─ Overhead: 5%
├─ Throughput: 95%
└─ Rating: 9/10 ✅

System-level (VM):
├─ Time: 10.3 seconds
├─ Overhead: 3%
├─ Throughput: 97%
└─ Rating: 9.5/10 ✅

Winner: System-level (less overhead) ✅


I/O-bound workload (Network server):

Native:
├─ Throughput: 10 Gbps
└─ Latency: 50 μs

Process-level (Java):
├─ Throughput: 9.5 Gbps (95%)
├─ Latency: 55 μs
└─ GC pauses: 10-50ms ⚠️

System-level (VM with virtio):
├─ Throughput: 8.5 Gbps (85%)
├─ Latency: 60 μs
└─ Consistent ✅

Winner: Process-level (higher throughput)


Memory-intensive workload:

Native:
├─ Memory: 1 GB
└─ Access time: 100 ns

Process-level (Java):
├─ Memory: 1.2 GB (+200 MB JVM)
├─ Access time: 100 ns
└─ GC overhead: 5-10% ⚠️

System-level (VM):
├─ Memory: 2 GB (+1 GB OS)
├─ Access time: 105 ns (EPT)
└─ Overhead: 5% ⚠️

Winner: Process-level (less memory)
```

---

**6. Use Case Suitability**
```
PROCESS-LEVEL VM best for:

✅ Application portability
   - Deploy same code on Windows/Linux/macOS
   - Example: Enterprise Java apps

✅ Managed runtime benefits
   - Automatic memory management
   - JIT optimization
   - Example: Web applications (Spring Boot)

✅ Sandboxing untrusted code
   - Security sandbox
   - Example: Browser (JavaScript V8)

✅ Microservices
   - Lightweight (50-100 MB)
   - Fast startup (1-2 sec)
   - Example: Serverless functions

❌ NOT suitable for:
   - Multiple OS requirements
   - Strong isolation needs
   - Legacy app support (different OS)


SYSTEM-LEVEL VM best for:

✅ OS diversity
   - Run Windows + Linux on same hardware
   - Example: Data centers

✅ Strong isolation
   - Security boundaries
   - Example: Multi-tenant cloud

✅ Full system snapshots
   - Backup/restore entire OS
   - Example: Development environments

✅ Hardware abstraction
   - Move VMs between physical servers
   - Example: Cloud computing (AWS EC2)

❌ NOT suitable for:
   - Ultra-low latency (<1ms)
   - Minimal overhead requirement
   - Lightweight deployments
```

---

**7. Điểm khác biệt cơ bản**
```
╔══════════════════════════════════════════════════╗
║      CORE DIFFERENCES                            ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  PROCESS-LEVEL VM:                               ║
║  ┌────────────────────────────────────────────┐ ║
║  │ Virtualizes: Instruction Set Architecture  │ ║
║  │ Granularity: Per-application               │ ║
║  │ Guest runs: Bytecode                       │ ║
║  │ Host provides: OS services                 │ ║
║  │ Examples: JVM, CLR, V8                     │ ║
║  └────────────────────────────────────────────┘ ║
║                                                  ║
║  SYSTEM-LEVEL VM:                                ║
║  ┌────────────────────────────────────────────┐ ║
║  │ Virtualizes: Entire hardware platform      │ ║
║  │ Granularity: Per-OS                        │ ║
║  │ Guest runs: Full OS + apps                 │ ║
║  │ Host provides: Hardware abstraction        │ ║
║  │ Examples: VMware ESXi, KVM, Xen            │ ║
║  └────────────────────────────────────────────┘ ║
║                                                  ║
║  KEY DISTINCTION:                                ║
║  Process-level: One OS, many runtimes           ║
║  System-level: Many OS, one hypervisor          ║
║                                                  ║
╚══════════════════════════════════════════════════╝