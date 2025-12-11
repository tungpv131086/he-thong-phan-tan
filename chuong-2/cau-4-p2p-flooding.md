CHƯƠNG 2 - CÂU 4
Đề bài:
Giả sử bạn xây dựng một ứng dụng chia sẻ tệp tin P2P không cấu trúc. Áp dụng cơ chế flooding, hãy mô tả cách tìm kiếm một tệp tin khi nút phát yêu cầu có TTL = 3. Phân tích tình huống: Nếu hệ thống có mật độ nút cao, việc chọn TTL = 3 có thể dẫn đến hệ quả gì về độ bao phủ và chi phí truyền thông?

BÀI GIẢI:
Phần 1: Kiến thức nền tảng về P2P không cấu trúc và Flooding
A. P2P không cấu trúc (Unstructured P2P)
Đặc điểm:

Không có cấu trúc tổ chức cố định như DHT (Chord, Pastry)
Nodes kết nối ngẫu nhiên với nhau tạo thành overlay network
Không có quy tắc về việc node nào lưu file nào
Ví dụ: Gnutella, early BitTorrent, Kazaa

Cấu trúc mạng:
        Node A ──── Node B
         │  \      /   │
         │   \    /    │
         │    \  /     │
        Node C ─── Node D ──── Node E
         │              │  \
         │              │   \
        Node F ──── Node G ─ Node H
Ưu điểm:

Dễ xây dựng, không cần thuật toán phức tạp
Linh hoạt, dễ thêm/bớt node
Chịu lỗi cao (không có single point of failure)

Nhược điểm:

Không đảm bảo tìm thấy file (ngay cả khi file tồn tại)
Chi phí tìm kiếm cao (phải hỏi nhiều nodes)
Không scale tốt với mạng lớn


B. Cơ chế Flooding
Định nghĩa:
Flooding là kỹ thuật tìm kiếm trong P2P không cấu trúc, trong đó query được broadcast đến tất cả các neighbors, và các neighbors tiếp tục forward đến neighbors của chúng.
Các thành phần chính:

Query Message gồm:

Query ID (unique identifier)
TTL (Time-To-Live): số hops tối đa
Search criteria (tên file, keyword)
Source node ID


TTL (Time-To-Live):

Giới hạn phạm vi tìm kiếm
Giảm 1 sau mỗi hop
Khi TTL = 0, message bị drop


Duplicate detection:

Mỗi node lưu cache các Query ID đã thấy
Tránh xử lý cùng một query nhiều lần




Phần 2: Mô tả chi tiết cơ chế Flooding với TTL = 3
Giả định:
Topology mạng P2P:
- Node S (Source): Node phát query
- Mỗi node có 3-4 neighbors
- File cần tìm: "movie.mp4"
- TTL khởi đầu: 3
Cấu trúc mạng ví dụ:
                Level 0 (TTL=3)
                    [S]
                    / \
                   /   \
Level 1 (TTL=2)  [A]   [B]
                 / \    / \
                /   \  /   \
Level 2 (TTL=1)[C] [D][E] [F]
               / |  | \ |\ | \
Level 3 (TTL=0)[G][H][I][J][K][L][M]

BƯỚC 1: Node S khởi tạo query (TTL = 3)
Node S thực hiện:
python# Pseudo-code tại Node S
def search_file(filename):
    query = {
        'query_id': generate_uuid(),  # e.g., "q-12345"
        'ttl': 3,
        'filename': "movie.mp4",
        'source': 'S',
        'hops': 0
    }
    
    # Lưu vào cache để tránh xử lý lại
    cache_query(query['query_id'])
    
    # Kiểm tra local storage
    if file_exists_locally(filename):
        return local_file_path
    
    # Forward đến tất cả neighbors
    for neighbor in get_neighbors():  # [A, B]
        send_query(neighbor, query)
    
    # Đợi QueryHit responses (timeout = 30s)
    wait_for_responses()
```

**Query được gửi:**
```
Query Message:
┌──────────────────────────────┐
│ Query ID:   q-12345          │
│ TTL:        3                │
│ Filename:   movie.mp4        │
│ Source:     S                │
│ Hops:       0                │
└──────────────────────────────┘

Node S gửi đến: [A, B]
```

**Trạng thái:**
```
Node S:
  ├─ Cache: {q-12345}
  ├─ Queries sent: 2 (to A, B)
  └─ Waiting for responses...

BƯỚC 2: Level 1 - Nodes A và B nhận query (TTL = 2)
Node A xử lý:
pythondef receive_query(query, from_node):
    query_id = query['query_id']
    
    # 1. Kiểm tra duplicate
    if query_id in query_cache:
        return  # Đã xử lý rồi, bỏ qua
    
    # 2. Thêm vào cache
    cache_query(query_id)
    
    # 3. Kiểm tra TTL
    if query['ttl'] <= 0:
        return  # Drop message
    
    # 4. Kiểm tra local storage
    if file_exists_locally(query['filename']):
        send_query_hit(query['source'], file_info)
        return
    
    # 5. Decrement TTL và forward
    query['ttl'] -= 1
    query['hops'] += 1
    
    # 6. Forward đến neighbors (trừ node gửi đến)
    for neighbor in get_neighbors():
        if neighbor != from_node:  # Không gửi lại cho người gửi
            send_query(neighbor, query)
```

**Node A thực hiện:**
```
Nhận query từ S:
  ├─ Query ID: q-12345
  ├─ TTL: 3 → Decrement to 2
  ├─ Check cache: Chưa thấy → Add to cache
  ├─ Check local: Không có file
  └─ Forward to neighbors: [C, D] (exclude S)
```

**Node B thực hiện tương tự:**
```
Nhận query từ S:
  ├─ TTL: 3 → Decrement to 2
  ├─ Check cache: Chưa thấy → Add to cache
  ├─ Check local: Không có file
  └─ Forward to neighbors: [E, F] (exclude S)
```

**Số messages tại Level 1:**
```
Messages sent:
- Node A → [C, D]: 2 messages
- Node B → [E, F]: 2 messages
Total: 4 messages
```

---

**BƯỚC 3: Level 2 - Nodes C, D, E, F nhận query (TTL = 1)**

**Node C xử lý:**
```
Nhận query từ A:
  ├─ Query ID: q-12345
  ├─ TTL: 2 → Decrement to 1
  ├─ Check cache: Chưa thấy → Add
  ├─ Check local: Không có file
  └─ Forward to neighbors: [G, H] (exclude A)
```

**Node D xử lý:**
```
Nhận query từ A:
  ├─ TTL: 2 → Decrement to 1
  ├─ Check cache: Chưa thấy → Add
  ├─ Check local: CÓ FILE! 🎯
  └─ Send QueryHit back to Source S
```

**QueryHit Response từ Node D:**
```
QueryHit Message:
┌──────────────────────────────┐
│ Query ID:   q-12345          │
│ Filename:   movie.mp4        │
│ File Size:  1.2 GB           │
│ Owner:      Node D           │
│ IP Address: 192.168.1.100    │
│ Port:       6346             │
└──────────────────────────────┘

Path: D → A → S
```

**Nodes E, F tiếp tục forward:**
```
Node E:
  └─ Forward to: [I, J]
  
Node F:
  └─ Forward to: [K, L]
```

**Số messages tại Level 2:**
```
Messages sent:
- Node C → [G, H]: 2
- Node D → [I]: 1 (và gửi QueryHit)
- Node E → [I, J]: 2
- Node F → [K, L]: 2
Total: 7 messages + 1 QueryHit
```

---

**BƯỚC 4: Level 3 - Nodes G, H, I, J, K, L nhận query (TTL = 0)**

**Nodes ở Level 3 xử lý:**
```
Node G:
  ├─ TTL: 1 → Decrement to 0
  ├─ Check cache: Chưa thấy → Add
  ├─ Check local: Không có file
  └─ TTL = 0 → DROP (không forward tiếp)

Tương tự cho H, I, J, K, L, M...
```

**Kết quả:**
```
Tất cả nodes ở Level 3:
- Kiểm tra local storage
- KHÔNG forward tiếp (vì TTL = 0)
- Nếu có file → Send QueryHit
```

**Số messages tại Level 3:**
```
Messages sent: 0 (TTL = 0, không forward)
Total checks: 7 nodes kiểm tra local storage
```

---

**BƯỚC 5: Node S nhận QueryHit và download**

**Node S nhận response:**
```
QueryHit từ Node D:
┌──────────────────────────────┐
│ File found: movie.mp4        │
│ Owner: Node D                │
│ IP: 192.168.1.100:6346       │
│ File size: 1.2 GB            │
└──────────────────────────────┘
Node S thực hiện download:
pythondef handle_query_hit(query_hit):
    # Hiển thị kết quả cho user
    display_search_result(query_hit)
    
    # Nếu user chọn download
    if user_confirms_download():
        # Kết nối trực tiếp đến Node D
        establish_connection(query_hit['ip'], query_hit['port'])
        
        # Download file qua HTTP/BitTorrent protocol
        download_file(query_hit['filename'])
```

**Download process:**
```
S ─────── Direct TCP Connection ───────> D
          (HTTP/BitTorrent)
          
S <────── File chunks (1.2 GB) ────────── D
```

---

#### **Phần 3: Tổng kết quá trình Flooding với TTL = 3**

**Sơ đồ hoàn chỉnh:**
```
                  [S] TTL=3 (Start)
                   │ └─ Forward to A, B
                   │
        ┌──────────┴──────────┐
        │                     │
       [A] TTL=2             [B] TTL=2
        │ └─ Forward         │ └─ Forward
    ┌───┴───┐           ┌───┴───┐
    │       │           │       │
   [C]     [D]🎯       [E]     [F]
  TTL=1   TTL=1       TTL=1   TTL=1
    │       │           │       │
  ┌─┴─┐   [I]       ┌──┴──┐  ┌─┴─┐
  │   │   TTL=0     │     │  │   │
 [G] [H]           [I]   [J][K] [L]
TTL=0 TTL=0       TTL=0 TTL=0 TTL=0

Legend:
🎯 = File found
│  = Query propagation
```

**Thống kê:**

| Level | TTL | Nodes visited | Messages sent | Nodes checked |
|-------|-----|---------------|---------------|---------------|
| 0 | 3 | 1 (S) | 2 | 1 |
| 1 | 2 | 2 (A, B) | 4 | 2 |
| 2 | 1 | 4 (C, D, E, F) | 7 | 4 |
| 3 | 0 | 7+ (G, H, I, J, K, L, M) | 0 | 7+ |
| **Total** | - | **14+** | **13** | **14+** |

**Công thức tổng quát:**

Với mạng có độ phân nhánh trung bình = k (mỗi node có k neighbors):
```
Tổng số nodes được visit với TTL = t:
N(t) = 1 + k + k² + k³ + ... + k^t
     = (k^(t+1) - 1) / (k - 1)

Với k = 2.5, TTL = 3:
N(3) = (2.5^4 - 1) / (2.5 - 1) 
     ≈ 26 nodes
```

---

#### **Phần 4: Phân tích hệ quả với mật độ nút cao**

**Tình huống: Hệ thống có mật độ nút cao**

Giả sử:
- Tổng số nodes trong mạng: N = 10,000 nodes
- Mỗi node có trung bình k = 10 neighbors
- TTL = 3

---

**A. Phân tích về Độ bao phủ (Coverage)**

**1. Số nodes được reach với TTL = 3:**
```
Level 0 (TTL=3): 1 node (source)
Level 1 (TTL=2): 10 nodes
Level 2 (TTL=1): 10 × 9 = 90 nodes (trừ duplicate)
Level 3 (TTL=0): 90 × 9 = 810 nodes

Thực tế (trừ overlapping): ~600-800 nodes
```

**Coverage ratio:**
```
Coverage = Nodes reached / Total nodes
         = 800 / 10,000
         = 8%
```

**Ý nghĩa:**
```
✅ Ưu điểm:
- Tìm kiếm nhanh (3 hops)
- Chi phí tương đối thấp trong mạng lớn

❌ Nhược điểm:
- Chỉ cover 8% mạng
- Nếu file ở 92% nodes còn lại → KHÔNG TÌM THẤY
- Success rate thấp với rare files
```

**2. Xác suất tìm thấy file:**

Giả sử file phổ biến có replication ratio = r (tỷ lệ nodes có file):
```
r = 0.01 (1%):  Prob(found) = 1 - (1-0.01)^800 ≈ 99.97% ✅
r = 0.001 (0.1%): Prob(found) = 1 - (1-0.001)^800 ≈ 55% ⚠️
r = 0.0001 (0.01%): Prob(found) = 1 - (1-0.0001)^800 ≈ 7.7% ❌
```

**Kết luận về coverage:**
```
TTL = 3 với mật độ cao:
✅ Tốt cho: Popular files (high replication)
❌ Kém cho: Rare files (low replication)

Để tăng coverage:
- Tăng TTL (4, 5, 6...)
- Hoặc dùng hybrid approach (expanding ring search)
```

---

**B. Phân tích về Chi phí truyền thông (Communication Cost)**

**1. Tổng số messages với k = 10, TTL = 3:**
```
Công thức:
Messages = k + k² + k³
         = 10 + 100 + 1,000
         = 1,110 messages PER QUERY
```

**So sánh với các giá trị TTL khác:**

| TTL | Nodes reached | Messages sent | Coverage (N=10k) |
|-----|---------------|---------------|------------------|
| 1 | ~10 | 10 | 0.1% |
| 2 | ~100 | 110 | 1% |
| 3 | ~800 | 1,110 | 8% |
| 4 | ~7,000 | 11,110 | 70% |
| 5 | ~63,000 | 111,110 | 630% (overlap) |

**Nhận xét:**
```
TTL tăng từ 3 → 4:
- Coverage tăng: 8% → 70% (+62%) ✅
- Messages tăng: 1,110 → 11,110 (×10) ❌

→ Trade-off rất lớn!
```

---

**2. Network congestion (Tắc nghẽn mạng)**

Với mật độ nút cao (k = 10):

**Message explosion:**
```
Query được phát sinh exponentially:
- Level 1: 10 messages
- Level 2: 100 messages
- Level 3: 1,000 messages

Trong 1 giây, nếu có 100 concurrent queries:
Total messages = 100 × 1,110 = 111,000 messages/second
```

**Bandwidth consumption:**
```
Giả sử mỗi query message = 500 bytes:
Bandwidth = 111,000 × 500 bytes
          = 55.5 MB/second
          = 444 Mbps

→ Có thể làm tắc nghẽn mạng!
```

**Duplicate processing overhead:**

Nodes ở "giao điểm" nhận nhiều bản copy:
```
       Node A
      /  |  \
     /   |   \
   [B]  [C]  [D]
     \   |   /
      \  |  /
       Node E ← Nhận query từ B, C, D (3 lần!)
       
Node E phải:
- Check duplicate: 3 times
- Potentially forward: 3 times (nếu chưa cache)
```

**CPU cost:**
```
Mỗi node xử lý:
- Cache lookup: O(1)
- Local file search: O(log n) hoặc O(1) với index
- Message forwarding: O(k) với k neighbors

Với 800 nodes visited:
Total CPU cycles = 800 × (cache + search + forward)
```

---

**3. False positives và duplicate responses:**

**Vấn đề:**
```
Nhiều nodes có cùng file → Multiple QueryHits về Source

Ví dụ: File "movie.mp4" có ở 50 nodes trong coverage area
→ Source nhận 50 QueryHit messages
→ Phải xử lý và filter
```

**Bandwidth waste:**
```
50 QueryHit × 1 KB = 50 KB
Nhân với 100 concurrent queries = 5 MB extra traffic
```

---

**C. Các vấn đề khác với TTL = 3 trong mạng mật độ cao**

**1. Hot spots (Điểm nóng):**
```
Nodes ở trung tâm mạng:
- Nhận và forward RẤT NHIỀU queries
- CPU và bandwidth overload
- Có thể crash hoặc disconnect

       [A]     [B]     [C]
         \     |     /
          \    |    /
            [HUB] ← Điểm nóng!
          /    |    \
         /     |     \
       [D]     [E]     [F]
```

**2. Free-riding problem:**
```
Một số users chỉ search mà không share files:
- Lợi dụng network để tìm kiếm
- Không đóng góp vào hệ thống
- Làm giảm availability của rare files
```

**3. Security concerns:**
```
❌ Query flooding attacks:
- Attacker gửi hàng ngàn fake queries
- Làm tê liệt network với TTL cao
- DDoS mạng P2P

❌ Index poisoning:
- Fake QueryHit responses
- Dẫn users tới malware

Phần 5: Giải pháp và cải tiến
A. Expanding Ring Search (Tìm kiếm vòng mở rộng)
Thay vì dùng TTL cố định, tăng dần:
pythondef expanding_ring_search(filename):
    ttl_levels = [1, 2, 3, 5, 7]  # Tăng dần
    
    for ttl in ttl_levels:
        results = flood_search(filename, ttl)
        
        if results:
            return results  # Tìm thấy, dừng lại
        
        sleep(2)  # Đợi trước khi tăng TTL
    
    return None  # Không tìm thấy
```

**Lợi ích:**
```
✅ Tìm popular files nhanh với TTL thấp
✅ Tăng dần coverage cho rare files
✅ Giảm traffic cho majority cases

B. Random Walk (Thay thế Flooding)
Thay vì broadcast, gửi k walkers ngẫu nhiên:
pythondef random_walk_search(filename, num_walkers=5, max_hops=20):
    for i in range(num_walkers):
        current_node = self
        
        for hop in range(max_hops):
            if current_node.has_file(filename):
                return current_node
            
            # Chọn random neighbor
            current_node = random.choice(current_node.neighbors)
    
    return None
```

**So sánh:**
```
Flooding (TTL=3): 1,110 messages
Random Walk (5 walkers × 20 hops): 100 messages

→ Giảm 91% traffic!
```

---

**C. Super-peer Architecture**

Hybrid P2P: Một số nodes làm super-peer:
```
Regular Peers          Super Peers
    [A]                    [S1]
    [B] ──connect──→       [S2] ←─── Exchange index
    [C]                    [S3]
    [D]                    [S4]

Query flow:
1. A gửi query → S1 (super-peer)
2. S1 check index (biết B, C, D có gì)
3. S1 query các super-peers khác
4. Trả kết quả cho A
```

**Lợi ích:**
```
✅ Giảm flooding exponential → Linear queries
✅ Faster search (super-peers có index)
❌ Phụ thuộc vào super-peers (single point of failure)

D. Bloom Filters for caching
Sử dụng Bloom filter để tóm tắt nội dung của neighbors:
pythonclass Node:
    def __init__(self):
        self.bloom_filter = BloomFilter(size=1000)
        
        # Thêm tất cả files vào bloom filter
        for file in self.local_files:
            self.bloom_filter.add(file)
    
    def query(self, filename):
        # Check bloom filter của neighbors trước
        candidates = []
        for neighbor in self.neighbors:
            if neighbor.bloom_filter.might_contain(filename):
                candidates.append(neighbor)
        
        # Chỉ gửi query đến candidates (filtered)
        for candidate in candidates:
            send_query(candidate, filename)
```

**Giảm messages:**
```
Trước: Gửi đến 10 neighbors
Sau: Bloom filter → Chỉ 2-3 candidates
→ Giảm 70% messages
```

---

#### **Phần 6: Kết luận và khuyến nghị**

**Tóm tắt phân tích TTL = 3 trong mạng mật độ cao:**

| Khía cạnh | Đánh giá | Chi tiết |
|-----------|----------|----------|
| **Coverage** | ⚠️ Trung bình | 8% mạng, tốt cho popular files |
| **Success rate** | ✅ Cao (popular) / ❌ Thấp (rare) | Phụ thuộc replication ratio |
| **Messages** | ❌ Cao | 1,110 messages/query |
| **Congestion** | ❌ Nghiêm trọng | Với concurrent queries cao |
| **Latency** | ✅ Thấp | 3 hops = nhanh |
| **Scalability** | ❌ Kém | Exponential message growth |

---

**Khuyến nghị:**

**Nên dùng TTL = 3 khi:**
```
✅ Mạng nhỏ-trung bình (< 1,000 nodes)
✅ Tìm kiếm popular files
✅ Network bandwidth dư thừa
✅ Low query frequency
```

**Nên tăng TTL hoặc dùng cải tiến khi:**
```
❌ Mạng lớn (> 10,000 nodes)
❌ Tìm kiếm rare files
❌ Network congestion
❌ High query frequency
```

**Giải pháp tốt nhất cho mạng mật độ cao:**
```
1. Expanding Ring Search (TTL = 1, 2, 3, 5...)
2. Kết hợp Random Walk (giảm traffic)
3. Super-peer architecture (giảm flooding)
4. Bloom filters (filter neighbors)
5. Caching QueryHit (avoid re-query)

Ví dụ thực tế:
Gnutella (early version):

Dùng flooding với TTL = 7
Message explosion nghiêm trọng
Network collapse với > 100,000 users

Gnutella 2.0 (cải tiến):

Super-peer architecture
TTL = 3-4 cho ultra-peers
Scalable đến millions users