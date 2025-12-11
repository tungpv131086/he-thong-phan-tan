CHƯƠNG 3 - CÂU 3
Đề bài:
Trong một hệ thống phân tán phục vụ đọc nhiều – ghi ít (read-heavy, write-light) như dịch vụ tập tin, bạn sẽ chọn mô hình đa luồng, đơn luồng, hay máy trạng thái hữu hạn (event-driven FSM) cho thành phần xử lý trên máy chủ? Hãy đánh giá dựa trên các tiêu chí: hiệu năng, khả năng mở rộng, độ phức tạp phát triển và khả năng chịu lỗi.

BÀI GIẢI:
Phần 1: Phân tích đặc điểm của hệ thống
A. Đặc điểm workload: Read-heavy, Write-light
Phân tích tỷ lệ:
Typical file service workload:
- Read operations: 95% (metadata lookup, file read)
- Write operations: 5% (file create, update, delete)

Request characteristics:
┌─────────────────────────────────────────┐
│ READ requests (95%):                    │
│ - High frequency: 10,000 req/sec       │
│ - Low latency required: <10ms           │
│ - I/O bound: Disk read, network send   │
│ - Can be cached                         │
│ - Independent (no conflicts)            │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ WRITE requests (5%):                    │
│ - Low frequency: 500 req/sec            │
│ - Can tolerate higher latency: <50ms    │
│ - Requires synchronization              │
│ - Must maintain consistency             │
│ - Potential conflicts                   │
└─────────────────────────────────────────┘
I/O pattern:
File read operation:
1. Lookup metadata (100μs - cache hit)
2. Read from disk (5ms - if not cached)
3. Send to client (2ms - network)
Total: ~7ms

→ I/O dominant (CPU idle 90% of time)
→ Opportunity for concurrency!

B. Yêu cầu hệ thống
Performance Requirements:
✓ Throughput: 10,000+ concurrent read requests
✓ Latency: P95 < 10ms for reads
✓ Scalability: Linear scaling with cores
✓ Resource efficient: Minimal memory per connection

Fault Tolerance:
✓ Handle client disconnections gracefully
✓ Recover from partial failures
✓ No data corruption under load

Development:
✓ Maintainable codebase
✓ Easy to debug
✓ Reasonable development time

Phần 2: Mô hình 1 - Single-threaded (Đơn luồng)
A. Kiến trúc Single-threaded
┌────────────────────────────────────────┐
│         FILE SERVER PROCESS            │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │   Main Thread (single)           │ │
│  │                                  │ │
│  │   while (1) {                    │ │
│  │     client = accept();           │ │
│  │     request = read(client);      │ │
│  │     response = process(request); │ │
│  │     write(client, response);     │ │
│  │     close(client);               │ │
│  │   }                              │ │
│  └──────────────────────────────────┘ │
└────────────────────────────────────────┘

Đặc điểm:
- Sequential processing
- One request at a time
- Blocking I/O
Code example:
c// Single-threaded file server
void single_threaded_server() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    bind(server_fd, ...);
    listen(server_fd, BACKLOG);
    
    while (1) {
        // Accept connection (BLOCKING)
        int client_fd = accept(server_fd, ...);
        
        // Read request (BLOCKING)
        Request req;
        read(client_fd, &req, sizeof(req));
        
        // Process request (BLOCKING on disk I/O)
        Response resp = handle_file_request(&req);
        
        // Send response (BLOCKING)
        write(client_fd, &resp, sizeof(resp));
        
        // Close connection
        close(client_fd);
        
        // Only now can we handle next request!
    }
}

Response handle_file_request(Request* req) {
    if (req->type == READ) {
        // Open file (disk I/O - BLOCKS)
        int fd = open(req->filename, O_RDONLY);
        
        // Read data (disk I/O - BLOCKS)
        char buffer[4096];
        read(fd, buffer, req->size);
        
        close(fd);
        
        return create_response(buffer);
    }
    // Similar for WRITE, DELETE, etc.
}
```

---

**B. Phân tích hiệu năng (Performance)**

**1. Throughput - RẤT THẤP ❌**
```
Timeline for 3 requests:

Time →
0ms   10ms  20ms  30ms  40ms  50ms  60ms
├─────┼─────┼─────┼─────┼─────┼─────┤
│ R1  │     │ R2  │     │ R3  │     │
│accpt│read │accpt│read │accpt│read │
│ ↓   │disk │ ↓   │disk │ ↓   │disk │
│proc │wait │proc │wait │proc │wait │
│send │     │send │     │send │     │
└─────┴─────┴─────┴─────┴─────┴─────┘

Each request: 20ms (10ms processing + 10ms wait)
Throughput: 1 request / 20ms = 50 requests/second

Required: 10,000 requests/second
Shortfall: 200x too slow! ❌
```

**2. CPU Utilization - RẤT KÉM ❌**
```
CPU Usage breakdown:

Processing:    10% (actual compute)
Disk I/O wait: 70% (CPU IDLE) ❌
Network wait:  15% (CPU IDLE) ❌
Other:         5%

Total CPU usage: ~10%
Wasted capacity: 90% ❌

Even on 8-core CPU:
- Only 1 core at 10% = 1.25% total utilization
- 7 cores completely idle
→ Massive waste!
```

**3. Latency - KÉM ❌**
```
Request latency:

Best case (no queue):
- Processing: 10ms
- Total: 10ms ✓

With load (10 requests waiting):
- Queue time: 10 × 20ms = 200ms
- Processing: 10ms
- Total: 210ms ❌
- P95 latency: >200ms (20x requirement!)
```

---

**C. Khả năng mở rộng (Scalability) - RẤT KÉM ❌**
```
Scaling analysis:

Add more CPU cores:
- Single thread uses only 1 core
- Other cores idle
- NO IMPROVEMENT ❌

Add more RAM:
- No effect (not memory-bound)
- NO IMPROVEMENT ❌

Add more servers:
- Linear cost increase
- Each server: 50 req/sec
- Need 200 servers for 10K req/sec!
- Cost: $$$$ ❌

Scalability score: 1/10 ❌

D. Độ phức tạp phát triển (Development Complexity) - TỐT ✅
c// Code rất đơn giản, dễ hiểu

void handle_request() {
    // Tuần tự, dễ theo dõi
    Step 1: Accept connection
    Step 2: Read request
    Step 3: Process
    Step 4: Send response
    Step 5: Close
    
    // No synchronization needed ✅
    // No race conditions ✅
    // No deadlocks ✅
}

// Debugging dễ dàng
// - Single execution path
// - gdb step-through works perfectly
// - printf debugging effective

Development time: 1 week ✅
Complexity: LOW (2/10) ✅
```

---

**E. Khả năng chịu lỗi (Fault Tolerance) - TRUNG BÌNH ⚠️**
```
Failure scenarios:

1. Client disconnect during processing:
   - Current request aborted
   - Can detect and handle
   - Other clients wait (queued)
   ⚠️ Impact: Delays for queued requests

2. Disk I/O error:
   - Easy to catch and handle
   - Return error to client
   - Continue with next request
   ✅ Good recovery

3. Process crash:
   - ALL connections lost ❌
   - No in-flight request recovery
   - Clients must retry
   ❌ Poor fault isolation

4. Resource exhaustion:
   - Memory leak → OOM
   - Entire server dies
   ❌ No graceful degradation

Fault tolerance score: 5/10 ⚠️
```

---

**F. Tóm tắt Single-threaded**

| Tiêu chí | Đánh giá | Điểm | Lý do |
|----------|----------|------|-------|
| **Performance** | ❌ Rất kém | 1/10 | 50 req/sec vs 10K required |
| **Scalability** | ❌ Rất kém | 1/10 | Không dùng được multicore |
| **Development** | ✅ Tốt | 9/10 | Code đơn giản, dễ debug |
| **Fault Tolerance** | ⚠️ TB | 5/10 | No isolation, but predictable |
| **OVERALL** | ❌ | **4/10** | **KHÔNG phù hợp** |

---

#### **Phần 3: Mô hình 2 - Multi-threaded (Đa luồng)**

**A. Kiến trúc Multi-threaded**

**Model 2a: Thread-per-request**
```
┌────────────────────────────────────────────────┐
│         FILE SERVER PROCESS                    │
│                                                │
│  Main Thread:                                  │
│  ┌──────────────────┐                         │
│  │ while (1) {      │                         │
│  │   client = accept()  ─────┐               │
│  │   spawn_thread(client) ───┼───┐           │
│  │ }                │         │   │           │
│  └──────────────────┘         │   │           │
│                                │   │           │
│  Worker Threads:               │   │           │
│  ┌────────────┐  ┌─────────────▼┐  ┌▼─────────┐│
│  │ Thread 1   │  │ Thread 2     │  │ Thread 3 ││
│  │ handle_req │  │ handle_req   │  │handle_req││
│  │  (client1) │  │  (client2)   │  │(client3) ││
│  └────────────┘  └──────────────┘  └──────────┘│
│       ...            ...               ...      │
│  (Thousands of threads)                        │
└────────────────────────────────────────────────┘
Code example:
c// Multi-threaded file server
void* worker_thread(void* arg) {
    int client_fd = *(int*)arg;
    free(arg);
    
    // Read request
    Request req;
    read(client_fd, &req, sizeof(req));
    
    // Process (this thread blocks on I/O, others continue!)
    Response resp = handle_file_request(&req);
    
    // Send response
    write(client_fd, &resp, sizeof(resp));
    
    close(client_fd);
    return NULL;
}

void multi_threaded_server() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    bind(server_fd, ...);
    listen(server_fd, BACKLOG);
    
    while (1) {
        int client_fd = accept(server_fd, ...);
        
        // Spawn new thread for each request
        pthread_t thread;
        int* arg = malloc(sizeof(int));
        *arg = client_fd;
        
        pthread_create(&thread, NULL, worker_thread, arg);
        pthread_detach(thread);  // Auto-cleanup
        
        // Main thread immediately available for next accept!
    }
}
```

**Model 2b: Thread pool (Better!)**
```
┌────────────────────────────────────────────────┐
│  Main Thread:                                  │
│  accept() → enqueue(request_queue)            │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │        Request Queue (bounded)           │ │
│  │  [R1] [R2] [R3] [R4] ... [RN]           │ │
│  └────┬─────┬─────┬─────┬───────────────────┘ │
│       │     │     │     │                     │
│  ┌────▼──┐ ┌▼────┐ ┌───▼┐ ┌─────┐           │
│  │Thread1│ │Thr 2│ │Thr3│ │ ... │           │
│  │Worker │ │Work │ │Work│ │ThrN │           │
│  └───────┘ └─────┘ └────┘ └─────┘           │
│    (Fixed pool size: e.g., 100 threads)      │
└────────────────────────────────────────────────┘
c// Thread pool implementation
typedef struct {
    pthread_t threads[POOL_SIZE];
    Queue* request_queue;
    pthread_mutex_t queue_lock;
    pthread_cond_t queue_cond;
    int shutdown;
} ThreadPool;

void* pool_worker(void* arg) {
    ThreadPool* pool = (ThreadPool*)arg;
    
    while (!pool->shutdown) {
        pthread_mutex_lock(&pool->queue_lock);
        
        // Wait for work
        while (queue_empty(pool->request_queue) && !pool->shutdown) {
            pthread_cond_wait(&pool->queue_cond, &pool->queue_lock);
        }
        
        if (pool->shutdown) {
            pthread_mutex_unlock(&pool->queue_lock);
            break;
        }
        
        // Dequeue request
        Request* req = queue_dequeue(pool->request_queue);
        pthread_mutex_unlock(&pool->queue_lock);
        
        // Process (no lock held!)
        handle_file_request(req);
    }
    
    return NULL;
}

void thread_pool_server() {
    ThreadPool pool;
    init_thread_pool(&pool, POOL_SIZE);
    
    int server_fd = socket(...);
    listen(server_fd, BACKLOG);
    
    while (1) {
        int client_fd = accept(server_fd, ...);
        
        Request* req = malloc(sizeof(Request));
        req->client_fd = client_fd;
        
        // Enqueue to pool
        pthread_mutex_lock(&pool.queue_lock);
        queue_enqueue(pool.request_queue, req);
        pthread_cond_signal(&pool.queue_cond);
        pthread_mutex_unlock(&pool.queue_lock);
    }
}
```

---

**B. Phân tích hiệu năng (Performance) - RẤT TỐT ✅**

**1. Throughput - CAO ✅**
```
Timeline with 100 threads:

Time →
0ms   10ms  20ms  30ms
├─────┼─────┼─────┤
Thread 1: [R1────]
Thread 2:   [R2────]
Thread 3:     [R3────]
...
Thread 100:         [R100──]

Concurrent processing: 100 requests
Each request: 10ms
Throughput: 100 / 0.01s = 10,000 req/sec ✅

Matches requirement! ✅
```

**2. CPU Utilization - TỐT ✅**
```
With 100 threads on 8-core CPU:

When thread blocks on I/O:
- OS schedules another thread
- Cores stay busy

CPU usage:
- Core 1: Thread1, Thread9, Thread17... (90% busy)
- Core 2: Thread2, Thread10, Thread18... (90% busy)
- ...
- Core 8: Thread8, Thread16, Thread24... (90% busy)

Total CPU utilization: ~90% ✅
Context switches: ~10,000/sec (manageable)

Effectively uses multicore! ✅
```

**3. Latency - TỐT ✅**
```
Request latency:

Low load (threads available):
- Queue time: ~0ms
- Processing: 10ms
- Total: 10ms ✅

High load (all threads busy):
- Queue time: ~10ms (wait for thread)
- Processing: 10ms
- Total: 20ms
- P95: <20ms ✅ (still good!)

Meets requirement: P95 < 10ms with headroom
```

---

**C. Khả năng mở rộng (Scalability) - TỐT ✅**
```
Vertical scaling (more cores):

1 core:  1,250 req/sec
2 cores: 2,500 req/sec (2x)
4 cores: 5,000 req/sec (4x)
8 cores: 10,000 req/sec (8x) ✅

Linear scaling! ✅

Horizontal scaling (more servers):

1 server:  10,000 req/sec
2 servers: 20,000 req/sec
4 servers: 40,000 req/sec

Also linear! ✅

Tuning thread pool size:
- Too small (10 threads): Underutilized, 1,250 req/sec
- Optimal (100 threads): Balanced, 10,000 req/sec ✅
- Too large (10,000 threads): Overhead, 9,000 req/sec ⚠️

Formula: threads = cores × (1 + wait_time/compute_time)
       = 8 × (1 + 7ms/3ms) ≈ 27 threads (minimum)
       = 100 threads (with headroom) ✅

Scalability score: 9/10 ✅

D. Độ phức tạp phát triển (Development Complexity) - TRUNG BÌNH ⚠️
c// Challenges:

1. Synchronization required ⚠️
pthread_mutex_t file_cache_lock;
pthread_rwlock_t metadata_lock;

// Must protect shared data structures
void update_metadata(File* file) {
    pthread_rwlock_wrlock(&metadata_lock);
    // Critical section
    file->last_modified = time(NULL);
    pthread_rwlock_unlock(&metadata_lock);
}

2. Race conditions possible ❌
// Bug: Lost update
// Thread 1:                    Thread 2:
count = get_count();       count = get_count();  // Both read 10
count++;                   count++;              // Both compute 11
set_count(count);          set_count(count);     // Both write 11
// Expected: 12, Actual: 11 ❌

3. Deadlocks possible ❌
// Thread 1:                    Thread 2:
lock(mutex_A);             lock(mutex_B);
lock(mutex_B); // WAIT     lock(mutex_A); // WAIT
// DEADLOCK! ❌

4. Memory leaks ⚠️
void* worker(void* arg) {
    Request* req = (Request*)arg;
    handle_request(req);
    // Forgot to free(req)! ❌
}

5. Thread lifecycle management
// Resource leak if not careful
pthread_t thread;
pthread_create(&thread, NULL, worker, data);
// Must: pthread_join() or pthread_detach()
// Otherwise: Thread resources not freed ❌
```

**Development complexity:**
```
Code lines: ~1,500 (vs 200 for single-threaded)
Development time: 4-6 weeks (vs 1 week)
Bug density: Higher (race conditions, deadlocks)
Debugging: Harder (non-deterministic bugs)
Testing: More complex (need concurrency tests)

Complexity score: 6/10 ⚠️
```

---

**E. Khả năng chịu lỗi (Fault Tolerance) - TỐT ✅**
```
Failure isolation:

1. Thread crash:
   ✅ Only that request fails
   ✅ Other threads unaffected
   ✅ Server continues
   
   Example:
   Thread 42: Segfault in handle_request()
   → Signal handler catches
   → Thread terminated
   → Threads 1-41, 43-100: Normal operation ✅

2. Client disconnect:
   ✅ Thread detects (EPIPE)
   ✅ Cleans up resources
   ✅ Returns to pool
   ✅ No impact on other clients

3. Resource exhaustion:
   ⚠️ Can be managed with bounded queues
   
   if (queue_full) {
       send_error(client, "503 Server Busy");
       // Graceful rejection ✅
   }

4. Deadlock detection:
   ⚠️ Possible with timeout mechanisms
   
   if (pthread_mutex_timedlock(&lock, &timeout) != 0) {
       // Timeout, possibly deadlock
       log_error("Potential deadlock");
       abort_request();
   }

Fault tolerance score: 8/10 ✅

F. Tối ưu hóa cho Read-heavy workload
c// Reader-Writer locks for read-heavy
pthread_rwlock_t cache_lock = PTHREAD_RWLOCK_INITIALIZER;

// Read path (concurrent, fast)
Response read_file(Request* req) {
    pthread_rwlock_rdlock(&cache_lock);  // Many readers OK!
    
    File* file = lookup_cache(req->filename);
    if (file) {
        Data data = file->data;
        pthread_rwlock_unlock(&cache_lock);
        return create_response(data);  // Fast! ✅
    }
    
    pthread_rwlock_unlock(&cache_lock);
    
    // Cache miss: Read from disk
    file = read_from_disk(req->filename);
    
    pthread_rwlock_wrlock(&cache_lock);  // Exclusive for write
    insert_cache(file);
    pthread_rwlock_unlock(&cache_lock);
    
    return create_response(file->data);
}

// Write path (exclusive, infrequent)
Response write_file(Request* req) {
    pthread_rwlock_wrlock(&cache_lock);  // Exclusive lock
    
    write_to_disk(req->filename, req->data);
    update_cache(req->filename, req->data);
    
    pthread_rwlock_unlock(&cache_lock);
    
    return create_response(OK);
}

Performance:
- 95% reads: Nearly zero lock contention ✅
- 5% writes: Occasional exclusive lock (acceptable)
- Overall: Excellent throughput ✅
```

---

**G. Tóm tắt Multi-threaded**

| Tiêu chí | Đánh giá | Điểm | Lý do |
|----------|----------|------|-------|
| **Performance** | ✅ Rất tốt | 9/10 | 10K req/sec, low latency |
| **Scalability** | ✅ Tốt | 9/10 | Linear scaling, multicore |
| **Development** | ⚠️ TB | 6/10 | Sync complexity, bugs |
| **Fault Tolerance** | ✅ Tốt | 8/10 | Good isolation, recovery |
| **OVERALL** | ✅ | **8/10** | **Rất phù hợp** ✅ |

---

#### **Phần 4: Mô hình 3 - Event-driven FSM (Non-blocking I/O)**

**A. Kiến trúc Event-driven**
```
┌────────────────────────────────────────────────┐
│         FILE SERVER PROCESS                    │
│                                                │
│  Main Event Loop (single thread):             │
│  ┌──────────────────────────────────────────┐ │
│  │ while (1) {                              │ │
│  │   events = epoll_wait(epoll_fd, ...);   │ │
│  │                                          │ │
│  │   for each event:                       │ │
│  │     if (event.type == ACCEPT)           │ │
│  │       handle_new_connection()           │ │
│  │     else if (event.type == READ)        │ │
│  │       handle_read_ready()               │ │
│  │     else if (event.type == WRITE)       │ │
│  │       handle_write_ready()              │ │
│  │     else if (event.type == DISK_IO)     │ │
│  │       handle_io_completion()            │ │
│  │ }                                        │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │   Async I/O Thread Pool (optional)       │ │
│  │   [Worker 1] [Worker 2] ... [Worker N]  │ │
│  │   (For disk I/O only)                    │ │
│  └──────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘

Key: Non-blocking I/O + Event multiplexing (epoll/kqueue)
Code example:
c// State machine for each connection
typedef enum {
    STATE_ACCEPTING,
    STATE_READING_REQUEST,
    STATE_PROCESSING,
    STATE_WRITING_RESPONSE,
    STATE_CLOSING
} ConnectionState;

typedef struct {
    int fd;
    ConnectionState state;
    Request req;
    Response resp;
    int bytes_read;
    int bytes_written;
} Connection;

// Main event loop
void event_driven_server() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    set_nonblocking(server_fd);
    bind(server_fd, ...);
    listen(server_fd, BACKLOG);
    
    // Create epoll instance
    int epoll_fd = epoll_create1(0);
    
    // Add server socket to epoll
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = server_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev);
    
    Connection* connections[MAX_CONNECTIONS];
    
    // Main event loop
    while (1) {
        struct epoll_event events[MAX_EVENTS];
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == server_fd) {
                // New connection
                handle_accept(epoll_fd, server_fd, connections);
            } else {
                // Existing connection has event
                Connection* conn = events[i].data.ptr;
                handle_connection_event(epoll_fd, conn, &events[i]);
            }
        }
    }
}

void handle_connection_event(int epoll_fd, Connection* conn, 
                             struct epoll_event* event) {
    switch (conn->state) {
        case STATE_READING_REQUEST:
            if (event->events & EPOLLIN) {
                // Socket readable
                int n = read(conn->fd, &conn->req + conn->bytes_read,
                           sizeof(Request) - conn->bytes_read);
                
                if (n > 0) {
                    conn->bytes_read += n;
                    
                    if (conn->bytes_read == sizeof(Request)) {
                        // Request complete
                        conn->state = STATE_PROCESSING;
                        
                        // Submit async I/O
                        submit_async_read(conn);
                    }
                } else if (n == 0) {
                    // EOF
                    close_connection(epoll_fd, conn);
                } else if (errno != EAGAIN) {
                    // Error
                    close_connection(epoll_fd, conn);
                }
            }
            break;
            
        case STATE_PROCESSING:
            // Async I/O completed (event from io_uring/AIO)
            if (event->events & EPOLLOUT) {
                conn->state = STATE_WRITING_RESPONSE;
                // Fall through to write
            }
            break;
            
        case STATE_WRITING_RESPONSE:
            if (event->events & EPOLLOUT) {
                // Socket writable
                int n = write(conn->fd, &conn->resp + conn->bytes_written,
                            sizeof(Response) - conn->bytes_written);
                
                if (n > 0) {
                    conn->bytes_written += n;
                    
                    if (conn->bytes_written == sizeof(Response)) {
                        // Response complete
                        conn->state = STATE_CLOSING;
                        close_connection(epoll_fd, conn);
                    }
                } else if (errno != EAGAIN) {
                    close_connection(epoll_fd, conn);
                }
            }
            break;
    }
}

// Async disk I/O (using io_uring or thread pool)
void submit_async_read(Connection* conn) {
    // Option 1: io_uring (Linux 5.1+)
    struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, file_fd, buffer, size, offset);
    io_uring_sqe_set_data(sqe, conn);
    io_uring_submit(&ring);
    
    // Option 2: Thread pool for disk I/O
    // submit_to_io_thread_pool(conn);
}
```

---

**B. Phân tích hiệu năng (Performance) - RẤT TỐT ✅**

**1. Throughput - RẤT CAO ✅**
```
Event loop performance:

Single thread handles ALL network I/O:
- epoll_wait: O(1) for ready events
- No context switches for network I/O
- Process multiple events per iteration

Timeline:
Time →
0μs   100μs  200μs  300μs  400μs
├─────┼──────┼──────┼──────┤
Event loop:
[R1 recv][R2 recv][R3 recv][R4 recv]...[R100 recv]
  ↓       ↓       ↓       ↓              ↓
[I/O]   [I/O]   [I/O]   [I/O]  ...    [I/O]
  ↓       ↓       ↓       ↓              ↓
[R1 send][R2 send][R3 send][R4 send]...[R100 send]

Process 100 requests in 400μs batch
Throughput: 100 / 0.4ms = 250,000 req/sec ✅✅✅

With async I/O thread pool (for disk):
Actual throughput: 50,000 req/sec
(Still 5x better than multi-threaded!)
```

**2. CPU Utilization - TỐT ✅**
```
CPU breakdown:

Main event thread:
- Network I/O handling: 30%
- Event processing: 20%
- Context management: 10%
- Total: 60% busy

Async I/O threads (4 threads):
- Disk I/O wait: 70%
- CPU: 30%

Overall:
- 1 main thread at 60% = 0.6 core
- 4 I/O threads at 30% = 1.2 cores
- Total: 1.8 cores used (out of 8)

Why so low?
- I/O bound workload (disk wait)
- Network I/O is async (no CPU during wait)

But: Can handle 50K req/sec with <2 cores! ✅
Efficiency: 25,000 req/sec per core ✅
(vs multi-threaded: 1,250 req/sec per core)

20x more efficient! ✅✅✅
```

**3. Latency - THẤP ✅**
```
Request latency breakdown:

Event-driven:
- Queue in event loop: <1ms
- Disk I/O (async): 5ms (concurrent)
- Network send: <1ms
- Total: ~7ms ✅

Multi-threaded (comparison):
- Queue wait: 1-5ms (depends on thread availability)
- Disk I/O: 5ms
- Network send: 1ms
- Total: ~10ms

Event-driven: 30% lower latency ✅

P95 latency:
- Event-driven: 8ms ✅
- Multi-threaded: 15ms
```

---

**C. Khả năng mở rộng (Scalability) - RẤT TỐT ✅**
```
C10K problem (10,000 concurrent connections):

Multi-threaded:
- 10,000 threads
- Memory: 10K × 8MB stack = 80 GB ❌
- Context switches: Excessive
- Infeasible ❌

Event-driven:
- 1 event loop thread
- 10,000 connections = 10K × 4KB state = 40 MB ✅
- No context switches for network I/O
- Feasible! ✅

C100K (100,000 connections):
- Memory: 400 MB (acceptable)
- Performance: Minimal degradation
- Event-driven wins! ✅

C1M (1 million connections):
- Memory: 4 GB (still OK on modern servers)
- Possible with event-driven ✅
- Impossible with threads ❌

Scalability score: 10/10 ✅✅✅
```

**Horizontal scaling:**
```
Event-driven server:

1 server: 50,000 req/sec (using 2 cores)
2 servers: 100,000 req/sec
4 servers: 200,000 req/sec

Linear scaling ✅

Cost efficiency:
- Can handle same load with fewer servers
- Lower infrastructure cost ✅

D. Độ phức tạp phát triển (Development Complexity) - CAO ❌
c// Challenge 1: Callback hell / State machine complexity

void handle_request(Connection* conn) {
    // Step 1: Read request
    read_async(conn->fd, &conn->req, sizeof(Request),
               on_request_read_complete, conn);
}

void on_request_read_complete(Connection* conn, int result) {
    if (result < 0) {
        handle_error(conn);
        return;
    }
    
    // Step 2: Read file
    read_file_async(conn->req.filename, conn->buffer, SIZE,
                   on_file_read_complete, conn);
}

void on_file_read_complete(Connection* conn, int result) {
    if (result < 0) {
        handle_error(conn);
        return;
    }
    
    // Step 3: Send response
    write_async(conn->fd, conn->buffer, result,
               on_response_write_complete, conn);
}

void on_response_write_complete(Connection* conn, int result) {
    // Step 4: Close
    close_connection(conn);
}

// → Deeply nested callbacks, hard to follow ❌
// → Error handling scattered everywhere ❌
// → Debugging nightmare ❌
Challenge 2: Shared state management
c// All state must be explicit in Connection struct
typedef struct {
    int fd;
    ConnectionState state;          // Explicit state
    Request req;
    Response resp;
    char buffer[BUFFER_SIZE];
    int bytes_read;                 // Partial I/O tracking
    int bytes_written;
    Timer* timeout;                 // Timeout handling
    void* user_data;                // Additional context
    // ... many more fields ...
} Connection;

// VS multi-threaded (implicit stack):
void handle_request(int client_fd) {
    Request req;               // On stack (automatic)
    read(client_fd, &req, ...); // Blocking, simple
    
    Response resp = process(req);
    write(client_fd, &resp, ...);
    // Stack automatically cleaned up
}

// Event-driven: Manual memory management ❌
// Must allocate, track, and free Connection ⚠️
Challenge 3: Error handling
c// Every callback must handle errors
void on_read(Connection* conn, int result) {
    if (result < 0) {
        if (errno == EAGAIN) {
            // Retry later (re-arm event)
            return;
        } else if (errno == ECONNRESET) {
            // Client disconnect
            cleanup_connection(conn);
            return;
        } else {
            // Other error
            log_error(errno);
            cleanup_connection(conn);
            return;
        }
    }
    
    // Normal path
    // ...
}

// Error handling interleaved with logic ❌
// Easy to miss edge cases ❌
```

**Development complexity:**
```
Code lines: ~3,000 (vs 1,500 multi-threaded)
Development time: 8-12 weeks ❌
Learning curve: Steep (event-driven paradigm)
Debugging: Very difficult (async, state machines)
Testing: Complex (must test all state transitions)

Common bugs:
- State machine errors
- Memory leaks (forgot to free Connection)
- Use-after-free (callback on closed connection)
- Race conditions (even with single thread!)

Complexity score: 3/10 ❌
```

---

**E. Khả năng chịu lỗi (Fault Tolerance) - TRUNG BÌNH ⚠️**
```
Pros:

1. Single-threaded main loop:
   ✅ No race conditions on event processing
   ✅ Deterministic behavior
   ✅ Easier to reason about

2. Graceful handling of slow clients:
   ✅ Non-blocking I/O → No blocking on slow write
   ✅ Can handle millions of connections
   ✅ Backpressure naturally handled

Cons:

1. Single point of failure:
   ❌ Main event loop crashes → Entire server dies
   ❌ Bug in event handler → All requests affected
   
   Example:
   void buggy_handler(Connection* conn) {
       int* ptr = NULL;
       *ptr = 42;  // Segfault!
       // → Entire server crashes ❌
   }

2. No isolation:
   ❌ CPU-intensive operation blocks event loop
   
   void expensive_operation() {
       for (int i = 0; i < 1000000000; i++) {
           compute();  // 10 seconds!
       }
       // → ALL connections frozen ❌
   }

3. Partial failure complexity:
   ⚠️ Connection in middle of state machine
   ⚠️ Must handle cleanup carefully
   
   if (conn->state == STATE_READING_REQUEST) {
       // Partial read, must cleanup buffer
       free(conn->req.partial_data);
   }

4. Cascade failures:
   ⚠️ Slow disk I/O → I/O thread pool exhausted
   ⚠️ → All requests queue up
   ⚠️ → Memory pressure
   ⚠️ → OOM

Fault tolerance score: 6/10 ⚠️

F. Optimization for Read-heavy workload
c// Event-driven is PERFECT for read-heavy!

// Cache in main thread (no locking needed!)
HashMap* file_cache;  // Single-threaded, no locks ✅

void handle_read_request(Connection* conn) {
    // Check cache (O(1), no lock)
    File* file = hashmap_get(file_cache, conn->req.filename);
    
    if (file) {
        // Cache hit: Serve immediately ✅
        conn->resp.data = file->data;
        conn->state = STATE_WRITING_RESPONSE;
        
        // No disk I/O needed!
        // Latency: <1ms ✅✅✅
    } else {
        // Cache miss: Async disk read
        submit_async_read(conn);
        // Other requests continue processing ✅
    }
}

// Write path: Rare, so can be slower
void handle_write_request(Connection* conn) {
    // Invalidate cache
    hashmap_remove(file_cache, conn->req.filename);
    
    // Submit async write
    submit_async_write(conn);
}

Performance for 95% reads:
- 90% cache hits: <1ms latency ✅✅✅
- 5% cache misses: 7ms latency (async I/O)
- 5% writes: 20ms latency (acceptable)

Average latency: 0.9×1ms + 0.05×7ms + 0.05×20ms
              = 0.9 + 0.35 + 1
              = 2.25ms ✅✅✅

MUCH better than multi-threaded (10ms) ✅

G. Tóm tắt Event-driven FSM
Tiêu chíĐánh giáĐiểmLý doPerformance✅✅ Xuất sắc10/1050K req/sec, 2ms latencyScalability✅✅ Xuất sắc10/10C1M possible, efficientDevelopment❌ Khó3/10Complex, callback hellFault Tolerance⚠️ TB6/10SPOF, but deterministicOVERALL✅7.25/10Phù hợp nếu có expertise
Phần 5: So sánh tổng hợp và Khuyến nghị
A. Bảng so sánh tổng quan
Tiêu chíSingle-threadedMulti-threadedEvent-driven FSMTrọng sốThroughput50 req/s ❌10,000 req/s ✅50,000 req/s ✅✅30%Latency (P95)210ms ❌15ms ✅8ms ✅✅25%CPU Efficiency10% ❌90% ✅60% ✅10%Memory per connN/A8 MB ❌4 KB ✅✅10%Max connections100 ❌1,000 ⚠️1,000,000 ✅✅15%Dev complexitySimple ✅✅Medium ⚠️Complex ❌5%Fault isolationNone ❌Good ✅Medium ⚠️5%WEIGHTED SCORE1.8/108.3/10 ✅9.1/10 ✅✅-

B. Chi tiết phân tích từng tiêu chí
1. Hiệu năng (Performance) - Trọng số 55%
Throughput:
Benchmark: Sustained load test

Single-threaded:
├─ Requests/sec: 50
├─ Bottleneck: Sequential processing
└─ Rating: 1/10 ❌

Multi-threaded (100 threads):
├─ Requests/sec: 10,000
├─ Bottleneck: Context switch overhead
├─ Rating: 8/10 ✅

Event-driven:
├─ Requests/sec: 50,000
├─ Bottleneck: Disk I/O (with io_uring)
└─ Rating: 10/10 ✅✅

Winner: Event-driven (5x better than multi-threaded)
Latency:
Latency distribution (milliseconds):

                P50   P95   P99   P99.9
Single-thread:   20   210   500   1000   ❌
Multi-thread:    10    15    25     50   ✅
Event-driven:     2     8    12     20   ✅✅

Latency variance:
Single-thread: High (depends on queue)
Multi-thread:  Medium (thread scheduling)
Event-driven:  Low (predictable event loop) ✅

Winner: Event-driven
CPU Efficiency:
CPU efficiency = Useful work / CPU cycles

Single-threaded:
- CPU usage: 10%
- Efficiency: 50 req/s per 10% = 5 req/s per 1% CPU
- Rating: 1/10

Multi-threaded:
- CPU usage: 90% (8 cores)
- Efficiency: 10K req/s per 720% = 13.9 req/s per 1% CPU
- Rating: 8/10

Event-driven:
- CPU usage: 60% (1.8 cores = 22.5%)
- Efficiency: 50K req/s per 22.5% = 2,222 req/s per 1% CPU ✅✅
- Rating: 10/10

Winner: Event-driven (160x better than multi-threaded!)
Memory Efficiency:
Memory per connection:

Single-threaded: ~0 (sequential, no state)
                but only handles 1 at a time ❌

Multi-threaded:
├─ Stack per thread: 8 MB
├─ 1,000 connections = 8 GB
├─ Practical limit: ~1,000 concurrent
└─ Rating: 4/10 ⚠️

Event-driven:
├─ State per connection: 4 KB
├─ 100,000 connections = 400 MB ✅
├─ Practical limit: 1,000,000+ concurrent
└─ Rating: 10/10 ✅✅

Winner: Event-driven (2,000x better!)

2. Khả năng mở rộng (Scalability) - Trọng số 15%
Vertical Scaling (thêm cores):
8-core CPU → 16-core CPU

Single-threaded:
├─ 50 req/s → 50 req/s (NO CHANGE ❌)
├─ Uses only 1 core
└─ Scaling factor: 0x

Multi-threaded:
├─ 10K req/s → 20K req/s ✅
├─ Linear scaling
└─ Scaling factor: 2x ✅

Event-driven:
├─ 50K req/s → 80K req/s ✅
├─ Near-linear (I/O bound)
└─ Scaling factor: 1.6x ✅

Winner: Multi-threaded (perfectly linear)
Horizontal Scaling (thêm servers):
All scale linearly:
├─ Single-threaded: 50 → 100 req/s (2 servers)
├─ Multi-threaded:  10K → 20K req/s
└─ Event-driven:    50K → 100K req/s ✅

But event-driven needs fewer servers:
Target: 100K req/s

Single-threaded: 2,000 servers! ❌ ($$$$$)
Multi-threaded:  10 servers ✅ ($$$)
Event-driven:    2 servers ✅✅ ($)

Winner: Event-driven (cost efficiency)
Connection Scalability:
Maximum concurrent connections:

Single-threaded:
├─ Sequential → Effectively 1 connection
└─ Rating: 1/10 ❌

Multi-threaded:
├─ Limit: ~10,000 threads (OS/memory limit)
├─ Real-world: ~1,000 threads (performance)
└─ Rating: 6/10 ⚠️

Event-driven:
├─ Limit: ~10,000,000 (fd limit)
├─ Real-world: 1,000,000+ (C1M problem solved!)
└─ Rating: 10/10 ✅✅

Winner: Event-driven (1,000x better!)

3. Độ phức tạp phát triển (Development Complexity) - Trọng số 5%
Code Complexity:
Lines of Code:

Single-threaded: 200 LOC
├─ Simple sequential flow
├─ Easy to understand
└─ Rating: 10/10 ✅✅

Multi-threaded: 1,500 LOC
├─ Synchronization code: 400 LOC
├─ Thread pool management: 300 LOC
├─ Error handling: 200 LOC
└─ Rating: 6/10 ⚠️

Event-driven: 3,000 LOC
├─ State machine: 800 LOC
├─ Callback management: 700 LOC
├─ Async I/O: 600 LOC
├─ Error handling everywhere: 400 LOC
└─ Rating: 3/10 ❌

Winner: Single-threaded (but unusable for production)
Development Time:
Team: 2 mid-level engineers

Single-threaded:
├─ Implementation: 1 week
├─ Testing: 1 week
├─ Total: 2 weeks ✅
└─ But: Cannot meet requirements ❌

Multi-threaded:
├─ Implementation: 4 weeks
├─ Thread-safety testing: 2 weeks
├─ Bug fixing (race conditions): 2 weeks
├─ Total: 8 weeks ✅
└─ Rating: 7/10

Event-driven:
├─ Learning curve: 2 weeks
├─ Implementation: 6 weeks
├─ State machine testing: 3 weeks
├─ Debugging async issues: 3 weeks
├─ Total: 14 weeks ⚠️
└─ Rating: 4/10

Winner: Multi-threaded (balanced)
Debugging Difficulty:
Bug: "Server sometimes returns wrong file"

Single-threaded:
├─ Add printf, reproduce easily
├─ gdb step-through works
├─ Time to fix: 1 hour ✅
└─ Rating: 10/10

Multi-threaded:
├─ Race condition: Hard to reproduce
├─ Use thread sanitizer (TSan)
├─ Helgrind, valgrind
├─ Time to fix: 1 day ⚠️
└─ Rating: 5/10

Event-driven:
├─ State machine bug: Hard to reproduce
├─ Async callback timing issue
├─ Need to add extensive logging
├─ Reconstruct state from logs
├─ Time to fix: 3 days ❌
└─ Rating: 3/10

Winner: Single-threaded (but impractical)
Practical winner: Multi-threaded
Maintainability:
Code maintainability (1 year later):

Single-threaded:
└─ Easy to modify ✅ (but still slow)

Multi-threaded:
├─ Synchronization complexity
├─ Need to understand locking strategy
├─ Potential to introduce deadlocks
└─ Moderate difficulty ⚠️

Event-driven:
├─ State machine interdependencies
├─ Callback chains hard to follow
├─ Adding new state is risky
└─ High difficulty ❌

Winner: Multi-threaded (best balance)

4. Khả năng chịu lỗi (Fault Tolerance) - Trọng số 5%
Failure Isolation:
Scenario: One request causes segfault

Single-threaded:
├─ Entire process crashes ❌
├─ ALL clients disconnected
├─ Requires process restart
└─ Impact: 100% of users ❌

Multi-threaded:
├─ Only faulting thread crashes ✅
├─ Other threads continue
├─ 99 out of 100 requests succeed
└─ Impact: 1% of users ✅

Event-driven:
├─ Main loop crashes ❌
├─ ALL clients disconnected
├─ (Unless segfault in I/O thread)
└─ Impact: 100% or 0% ⚠️

Winner: Multi-threaded
Graceful Degradation:
Scenario: High load (2x normal)

Single-threaded:
├─ Queue grows indefinitely
├─ Latency increases linearly
├─ Eventually OOM ❌
└─ Rating: 2/10

Multi-threaded:
├─ Bounded thread pool
├─ Request queue with limit
├─ Reject with 503 when full ✅
├─ Existing requests still processed
└─ Rating: 8/10 ✅

Event-driven:
├─ Can accept many connections
├─ But I/O thread pool saturated
├─ All requests slow ⚠️
├─ Need backpressure mechanism
└─ Rating: 6/10 ⚠️

Winner: Multi-threaded
Recovery from Failures:
Client disconnect mid-request:

Single-threaded:
├─ Detect on next I/O
├─ Abort current request
├─ Continue with next
└─ Clean recovery ✅

Multi-threaded:
├─ Thread detects EPIPE
├─ Cleanup resources
├─ Thread returns to pool
└─ Clean recovery ✅

Event-driven:
├─ Event: EPOLLHUP
├─ Must cleanup state machine
├─ Free allocated memory
├─ More complex but works ✅
└─ Clean recovery ✅

Winner: Tie (all handle well)
Monitoring and Observability:
Health check and metrics:

Single-threaded:
├─ Simple state
├─ Easy to monitor
├─ Metrics: Current request
└─ Rating: 8/10

Multi-threaded:
├─ Per-thread metrics
├─ Lock contention stats
├─ Thread pool utilization
└─ Rating: 9/10 ✅

Event-driven:
├─ Event loop latency
├─ Connection state distribution
├─ Async I/O queue depth
├─ More metrics needed
└─ Rating: 7/10

Winner: Multi-threaded

C. Phân tích theo workload characteristics
Read-heavy workload (95% reads):
Multi-threaded advantages:
✅ Reader-writer locks allow concurrent reads
✅ Minimal lock contention
✅ Simple to implement

Code:
pthread_rwlock_t cache_lock;

// 95% of requests (CONCURRENT)
pthread_rwlock_rdlock(&cache_lock);
data = cache_get(key);
pthread_rwlock_unlock(&cache_lock);

// 5% of requests (EXCLUSIVE)
pthread_rwlock_wrlock(&cache_lock);
cache_put(key, data);
pthread_rwlock_unlock(&cache_lock);

Performance: Excellent ✅
Event-driven advantages:
✅✅ NO LOCKS needed (single-threaded cache)
✅✅ Zero contention
✅✅ Maximum throughput

Code:
// Global cache (no locks!)
HashMap cache;

// 95% reads
data = hashmap_get(&cache, key);  // O(1), no lock ✅

// 5% writes
hashmap_put(&cache, key, data);   // O(1), no lock ✅

Performance: Outstanding ✅✅
Latency: <1ms for cache hits ✅✅
Winner for read-heavy: Event-driven ✅✅

Write-light workload (5% writes):
Multi-threaded:
├─ Writer lock acquired infrequently (5%)
├─ Minimal blocking of readers
├─ Acceptable overhead
└─ Rating: 8/10 ✅

Event-driven:
├─ No lock overhead ever
├─ Writes go to async I/O queue
├─ Readers continue unaffected
└─ Rating: 10/10 ✅✅

Winner: Event-driven

I/O-bound workload (file service):
Characteristics:
- 70% time in disk I/O
- 20% time in network I/O
- 10% time in CPU processing

Multi-threaded:
├─ Threads block on I/O
├─ OS schedules other threads
├─ CPU utilized during I/O wait ✅
└─ Rating: 8/10

Event-driven:
├─ Non-blocking I/O
├─ Async I/O with io_uring
├─ CPU processes other requests during I/O ✅✅
├─ Zero context switch overhead
└─ Rating: 10/10 ✅✅

Winner: Event-driven

D. Trade-offs Analysis
Multi-threaded trade-offs:
PROS:
✅ Good performance (10K req/s)
✅ Simple mental model (sequential per thread)
✅ Uses multicore effectively
✅ Good fault isolation
✅ Moderate development complexity
✅ Standard paradigm (many developers know)
✅ Good debugging tools (gdb, TSan)

CONS:
❌ Context switch overhead (~5μs per switch)
❌ Memory per connection (8 MB)
❌ Limited connections (~1,000 practical)
❌ Lock contention possible
❌ Race conditions possible
❌ Deadlocks possible

BEST FOR:
✓ Teams with thread programming experience
✓ Requirements: 1K-10K concurrent connections
✓ Mixed workloads (not purely I/O bound)
✓ Need quick time-to-market
✓ Standard enterprise applications
Event-driven trade-offs:
PROS:
✅✅ Excellent performance (50K req/s)
✅✅ Ultra-low memory (4 KB per conn)
✅✅ Massive connections (1M+)
✅✅ No lock overhead for single-threaded cache
✅✅ No context switch for network I/O
✅ Deterministic behavior
✅ Cache-friendly (single thread)

CONS:
❌❌ High development complexity
❌❌ Callback hell / state machine complexity
❌❌ Difficult debugging
❌❌ Steep learning curve
❌ Single point of failure
❌ Must be careful with blocking operations
❌ Fewer developers with expertise

BEST FOR:
✓ High-performance requirements (>20K req/s)
✓ Massive concurrency (>10K connections)
✓ I/O-bound workloads
✓ Read-heavy workloads
✓ Expert team with async programming experience
✓ Long-term project (amortize development cost)

E. Khuyến nghị cho File Service (Read-heavy)
Phân tích cụ thể:
File Service Requirements:
├─ Workload: 95% reads, 5% writes ✅ Read-heavy
├─ Expected load: 10K+ req/s
├─ Latency requirement: P95 < 10ms
├─ Concurrent connections: 10K-100K
└─ Development resources: Medium-sized team
KHUYẾN NGHỊ CHÍNH:
🏆 CHỌN MULTI-THREADED (Thread Pool)
Lý do:
1. PERFORMANCE: Đáp ứng yêu cầu ✅
   ├─ 10K req/s: Achievable
   ├─ P95 < 10ms: Achievable
   ├─ Reader-writer locks: Minimal contention
   └─ Can optimize for 50K req/s if needed

2. SCALABILITY: Đủ tốt ✅
   ├─ 10K connections: Comfortable
   ├─ 100K connections: Possible with tuning
   ├─ Linear vertical scaling
   └─ Linear horizontal scaling

3. DEVELOPMENT: Reasonable ✅✅
   ├─ 8 weeks development time
   ├─ Standard paradigm
   ├─ Good tooling support
   ├─ Easier to hire developers
   └─ Lower risk

4. MAINTENANCE: Manageable ✅
   ├─ Moderate complexity
   ├─ Standard debugging tools
   ├─ Well-understood patterns
   └─ Good documentation available

5. FAULT TOLERANCE: Good ✅
   ├─ Thread isolation
   ├─ Graceful degradation
   └─ Easy monitoring

Kiến trúc đề xuất (Multi-threaded):
c// Recommended architecture

┌─────────────────────────────────────────────┐
│          FILE SERVER (Multi-threaded)       │
│                                             │
│  Main Thread:                               │
│  ├─ Accept connections                      │
│  └─ Enqueue to request queue               │
│                                             │
│  Worker Thread Pool (100 threads):          │
│  ├─ Reader threads (80): Handle reads       │
│  │   └─ pthread_rwlock_rdlock()            │
│  ├─ Writer threads (20): Handle writes      │
│  │   └─ pthread_rwlock_wrlock()            │
│  └─ Async I/O helpers (optional)           │
│                                             │
│  Shared Data Structures:                    │
│  ├─ File metadata cache (RW-locked)        │
│  ├─ LRU cache (RW-locked)                  │
│  └─ Request queue (mutex-protected)        │
└─────────────────────────────────────────────┘

Optimizations:
1. Reader-writer locks for cache
2. Lock-free queue (if needed)
3. Per-core caching (advanced)
4. Async I/O with io_uring (optional)
c// Implementation sketch
typedef struct {
    pthread_rwlock_t lock;
    HashMap* metadata_cache;
    HashMap* data_cache;
    LRUCache* lru;
} SharedState;

void* reader_thread(void* arg) {
    while (1) {
        Request* req = dequeue();
        
        // Read path (shared lock)
        pthread_rwlock_rdlock(&state->lock);
        
        File* file = cache_get(state->data_cache, req->filename);
        
        pthread_rwlock_unlock(&state->lock);
        
        if (!file) {
            // Cache miss: read from disk
            file = read_from_disk(req->filename);
            
            // Update cache (exclusive lock)
            pthread_rwlock_wrlock(&state->lock);
            cache_put(state->data_cache, req->filename, file);
            pthread_rwlock_unlock(&state->lock);
        }
        
        send_response(req->client_fd, file->data);
    }
}

void* writer_thread(void* arg) {
    while (1) {
        Request* req = dequeue();
        
        // Write path (exclusive lock)
        pthread_rwlock_wrlock(&state->lock);
        
        cache_invalidate(state->data_cache, req->filename);
        write_to_disk(req->filename, req->data);
        
        pthread_rwlock_unlock(&state->lock);
        
        send_response(req->client_fd, OK);
    }
}
```

**Performance estimate:**
```
Configuration:
├─ Thread pool: 100 threads (80 readers, 20 writers)
├─ 8-core CPU
├─ Cache hit rate: 90%

Expected performance:
├─ Throughput: 12,000 req/s ✅
│   ├─ 90% cache hits: <1ms (10,800 req/s)
│   └─ 10% disk reads: 10ms (1,200 req/s)
├─ Latency P95: 8ms ✅
├─ Max connections: 10,000 ✅
└─ CPU utilization: 85% ✅

Meets all requirements! ✅✅✅
```

---

**KHI NÀO NÊN CHỌN EVENT-DRIVEN:**
```
Choose event-driven if:

1. SCALE requirements:
   ✓ Need >50K req/s single server
   ✓ Need >100K concurrent connections
   ✓ Infrastructure cost is critical

2. TEAM capabilities:
   ✓ Have expert async programmers
   ✓ Long development timeline acceptable
   ✓ Complex debugging not a concern

3. WORKLOAD characteristics:
   ✓ Pure I/O bound (>90%)
   ✓ Uniform request patterns
   ✓ No CPU-intensive operations

4. PROVEN architecture:
   ✓ Using battle-tested framework (libuv, tokio)
   ✓ Not building from scratch
   ✓ Reference implementations available

Example: nginx, Redis, Node.js
→ Already using event-driven successfully ✅
```

---

**HYBRID APPROACH (Advanced):**
```
Best of both worlds:

┌────────────────────────────────────────┐
│  Event-driven Main Loop                │
│  ├─ Handle network I/O (non-blocking)  │
│  └─ Fast path: Serve from cache       │
│                                        │
│  Thread Pool (for slow path):         │
│  ├─ Disk I/O operations                │
│  ├─ CPU-intensive processing           │
│  └─ Blocking operations                │
└────────────────────────────────────────┘

Pros:
✅ High performance (event-driven network)
✅ No blocking (thread pool for disk)
✅ Simpler than pure event-driven

Cons:
❌ Even more complex
❌ Requires careful coordination

Example: Haproxy, modern Nginx
```

---

#### **F. Bảng khuyến nghị cuối cùng**

| Scenario | Recommended | Alternative | Notes |
|----------|-------------|-------------|-------|
| **Standard file service** | Multi-threaded ✅ | Event-driven | Balanced approach |
| **High load (>50K req/s)** | Event-driven ✅ | Multi-threaded + scale | If expertise available |
| **Massive connections (>100K)** | Event-driven ✅✅ | - | Only viable option |
| **Small team, fast delivery** | Multi-threaded ✅✅ | - | Lower risk |
| **Read-heavy + caching** | Multi-threaded ✅ | Event-driven | RW-locks work well |
| **Mixed workload** | Multi-threaded ✅ | - | More flexible |
| **Expert team** | Event-driven ✅ | Hybrid | Best performance |

---

### **TÓM TẮT CUỐI CÙNG:**
```
╔══════════════════════════════════════════════════════╗
║  KHUYẾN NGHỊ CHO FILE SERVICE (READ-HEAVY)          ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  🏆 CHỌN: MULTI-THREADED (Thread Pool)              ║
║                                                      ║
║  LÝ DO:                                              ║
║  ✅ Đáp ứng performance requirements                ║
║  ✅ Reasonable development complexity               ║
║  ✅ Good fault tolerance                            ║
║  ✅ Proven, well-understood approach                ║
║  ✅ Standard tooling and debugging                  ║
║  ✅ Optimal cho read-heavy workload                 ║
║                                                      ║
║  ĐIỂM MẠNH:                                          ║
║  • 10K+ req/s throughput                            ║
║  • Reader-writer locks: minimal contention          ║
║  • Thread isolation: good fault tolerance           ║
║  • 8 weeks development: reasonable timeline         ║
║                                                      ║
║  KHI NÀO UPGRADE LÊN EVENT-DRIVEN:                  ║
║  • Khi cần scale >50K req/s                         ║
║  • Khi có expert team                               ║
║  • Sau khi profile và tối ưu multi-threaded         ║
║                                                      ║
╚══════════════════════════════════════════════════════╝