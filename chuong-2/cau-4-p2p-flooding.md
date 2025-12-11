# Câu 4: P2P Flooding Search với TTL

> **Chương:** 2 - Kiến trúc Hệ thống Phân tán  
> **Độ khó:** ⭐⭐⭐ (Trung bình)  
> **Thời gian đọc:** ~20 phút

---

## 📋 Mục lục

- [Đề bài](#đề-bài)
- [Phần 1: Cơ chế Flooding](#phần-1-cơ-chế-flooding)
- [Phần 2: Phân tích trong mạng mật độ cao](#phần-2-phân-tích-trong-mạng-mật-độ-cao)
- [Phần 3: Trade-offs và Giải pháp](#phần-3-trade-offs-và-giải-pháp)
- [Tóm tắt](#tóm-tắt)

---

## 📋 Đề bài

Trong mạng P2P sử dụng **flooding** để tìm kiếm tài nguyên, một query được gửi với **TTL = 3** (time-to-live).

**Giả sử:**
- Mỗi node có trung bình **k = 10 neighbors** (degree cao)
- Mạng có **N = 10,000 nodes**

**Yêu cầu:**

1. Ước lượng số **nodes** và **messages** được truyền trong quá trình flooding
2. Phân tích **trade-off** giữa:
   - Coverage (độ phủ mạng)
   - Message overhead (lượng thông điệp)
3. Đề xuất các biện pháp giảm overhead mà vẫn giữ hiệu quả tìm kiếm

---

## 💡 Bài giải

### Phần 1: Cơ chế Flooding

#### A. Thuật toán Flooding cơ bản
```
FLOODING ALGORITHM:
═══════════════════════════════════════════════

1. Node initiates query with TTL
2. Broadcast to all neighbors
3. Each neighbor:
   a. Decrements TTL
   b. If TTL > 0 AND not seen before:
      - Forward to all neighbors
   c. If has resource:
      - Send response back
4. Repeat until TTL = 0
```

**Pseudocode:**
```python
class Node:
    def __init__(self, node_id, neighbors):
        self.id = node_id
        self.neighbors = neighbors  # List of connected nodes
        self.seen_queries = set()   # Track processed queries
    
    def search(self, query_id, keyword, ttl, sender=None):
        """
        Flooding search algorithm
        """
        # Check if already processed this query
        if query_id in self.seen_queries:
            return  # Drop duplicate
        
        # Mark as seen
        self.seen_queries.add(query_id)
        
        print(f"Node {self.id}: Received query {query_id}, TTL={ttl}")
        
        # Check if this node has the resource
        if self.has_resource(keyword):
            print(f"Node {self.id}: FOUND! Sending response back")
            self.send_response(query_id, sender)
            return
        
        # If TTL expired, stop
        if ttl <= 0:
            print(f"Node {self.id}: TTL expired, stop")
            return
        
        # Forward to all neighbors (except sender)
        for neighbor in self.neighbors:
            if neighbor != sender:
                neighbor.search(query_id, keyword, ttl - 1, self)
```

#### B. Visualization với TTL = 3
```
FLOODING PROPAGATION (k=10 neighbors per node):
══════════════════════════════════════════════════

Level 0 (TTL=3): Starting node
              ●
              │ Query initiated
              └─ Broadcasts to 10 neighbors

Level 1 (TTL=2): First hop
        ┌─────┬─────┬───...───┬─────┐
        ●     ●     ●         ●     ●  (10 nodes)
        │ Each broadcasts to 9 neighbors
        │ (excluding sender)

Level 2 (TTL=1): Second hop
    ┌───┼───┬...
    ●   ●   ●  ...              (90 nodes)
    │ Each broadcasts to 9 neighbors

Level 3 (TTL=0): Third hop
  ●●●●● ...                     (810 nodes)
  │ TTL expired, stop

Total nodes reached: 1 + 10 + 90 + 810 = 911 nodes
```

#### C. Tính toán chính xác

**Số nodes được visit:**
```
Formula: Nodes(TTL) = 1 + k + k(k-1) + k(k-1)² + ... + k(k-1)^(TTL-1)

Với k=10, TTL=3:

Level 0: 1 node (starting node)
Level 1: k = 10 nodes
Level 2: k × (k-1) = 10 × 9 = 90 nodes
Level 3: k × (k-1)² = 10 × 81 = 810 nodes

Total: 1 + 10 + 90 + 810 = 911 nodes ✅

As percentage of network:
911 / 10,000 = 9.11% coverage
```

**Số messages được gửi:**
```
Messages = Edges traversed

Level 0 → Level 1: 10 messages (to 10 neighbors)
Level 1 → Level 2: 10 × 9 = 90 messages
Level 2 → Level 3: 90 × 9 = 810 messages

Total messages: 10 + 90 + 810 = 910 messages

Alternative formula:
Messages = Nodes - 1 (tree structure)
         = 911 - 1 = 910 ✅

Note: With duplicates (if not filtered):
Real messages could be 2-3x higher!
```

---

### Phần 2: Phân tích trong mạng mật độ cao

#### A. Impact of High Degree (k=10)
```
╔══════════════════════════════════════════════════╗
║        FLOODING IN HIGH-DEGREE NETWORK           ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  Metric              │ Value      │ Impact       ║
║ ═════════════════════╪════════════╪════════════  ║
║                                                  ║
║  Nodes visited       │    911     │ Good ✅      ║
║  Coverage            │   9.11%    │ Limited ⚠️   ║
║  Messages sent       │    910     │ High ⚠️      ║
║  Duplicate msgs      │  ~1,800    │ Very high ❌ ║
║  Network bandwidth   │  ~3.6 MB   │ Expensive ❌ ║
║                                                  ║
╚══════════════════════════════════════════════════╝

Assumptions:
- Message size: 4 KB (query + metadata)
- 910 messages × 4 KB = 3,640 KB ≈ 3.6 MB
- With duplicates: ~7.2 MB per query!
```

#### B. Success Rate Analysis

**Probability of finding resource:**
```python
def success_probability(replication_rate, coverage):
    """
    P(success) = 1 - P(miss all replicas)
               = 1 - (1 - r)^n
    
    r: replication rate (% nodes with resource)
    n: nodes visited
    """
    return 1 - (1 - replication_rate) ** coverage

# Calculate for different replication rates
replication = [0.001, 0.01, 0.1]  # 0.1%, 1%, 10%
nodes_visited = 911

for r in replication:
    p_success = success_probability(r, nodes_visited)
    print(f"Replication {r*100}%: Success rate = {p_success*100:.2f}%")
```

**Results:**
```
╔═══════════════════════════════════════════════╗
║      SUCCESS RATE vs REPLICATION              ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Replication │ Nodes   │ Success  │ Comment  ║
║  Rate        │ Visited │ Rate     │          ║
║ ═════════════╪═════════╪══════════╪═════════ ║
║              │         │          │          ║
║  0.01%       │   911   │   9.4%   │ Poor ❌  ║
║  (rare)      │         │          │          ║
║              │         │          │          ║
║  0.1%        │   911   │  60.0%   │ OK ⚠️    ║
║  (uncommon)  │         │          │          ║
║              │         │          │          ║
║  1%          │   911   │  99.99%  │ Good ✅  ║
║  (common)    │         │          │          ║
║              │         │          │          ║
║  10%         │   911   │ >99.99%  │ Overkill ║
║  (popular)   │         │          │          ║
║              │         │          │          ║
╚═══════════════════════════════════════════════╝

Conclusion:
- For popular files (r ≥ 1%): High success ✅
- For rare files (r < 0.1%): Poor success ❌
- TTL=3 insufficient for rare content!
```

#### C. Network Congestion
```
Scenario: 100 concurrent queries

Per query: 910 messages
Total: 100 × 910 = 91,000 messages

If each message = 4 KB:
Bandwidth = 91,000 × 4 KB = 364 MB
Time span = ~1 second
Rate = 364 MB/s = 2.9 Gbps 🔥

Impact:
❌ Network saturation
❌ Hub nodes overloaded
❌ High latency for all traffic
❌ Packet loss likely

Hub node (high degree):
- Receives queries from ALL neighbors
- Must forward to ALL neighbors
- Processing: 10 incoming × 9 outgoing = 90 messages
- If 100 concurrent queries: 9,000 messages/sec ❌
```

---

### Phần 3: Trade-offs và Giải pháp

#### A. TTL Trade-off Analysis
```
╔═══════════════════════════════════════════════════╗
║           TTL vs COVERAGE vs OVERHEAD             ║
╠═══════════════════════════════════════════════════╣
║                                                   ║
║  TTL │ Nodes    │ Messages  │ Coverage │ BW      ║
║      │ Visited  │           │          │         ║
║ ═════╪══════════╪═══════════╪══════════╪════════ ║
║      │          │           │          │         ║
║   1  │      11  │       10  │   0.1%   │  40 KB  ║
║   2  │     101  │      100  │   1.0%   │ 400 KB  ║
║   3  │     911  │      910  │   9.1%   │ 3.6 MB  ║
║   4  │   8,191  │    8,190  │  81.9%   │  32 MB  ║
║   5  │  73,711  │   73,710  │ 737%❗   │ 287 MB  ║
║      │          │           │ (overlap)│         ║
║                                                   ║
╚═══════════════════════════════════════════════════╝

Observations:
- TTL 3→4: Coverage 9× but overhead 9× ⚠️
- TTL 4→5: Exceeds network size (duplicates)
- Sweet spot: TTL = 3-4 for N=10K
```

**Mathematical relationship:**
```
Messages(TTL) ≈ k × (k-1)^TTL

For k=10:
TTL=1: 10
TTL=2: 10 × 9¹ = 90
TTL=3: 10 × 9² = 810
TTL=4: 10 × 9³ = 7,290

Exponential growth! 🚀
```

#### B. Biện pháp giảm overhead

**1. Expanding Ring Search**
```
Strategy: Start with low TTL, increase if not found

Step 1: Search with TTL=1
        ↓ Not found
Step 2: Search with TTL=2
        ↓ Not found
Step 3: Search with TTL=3
        ↓ Found! ✅

Total messages:
- If found at TTL=1: 10 messages ✅
- If found at TTL=2: 10 + 90 = 100 messages ✅
- If found at TTL=3: 10 + 90 + 810 = 910 messages

Average case (for popular files):
Much better than always using TTL=3! ✅

Code:
def expanding_ring_search(keyword):
    for ttl in [1, 2, 3, 5, 7]:  # Progressive TTLs
        result = flood_search(keyword, ttl)
        if result:
            return result
        time.sleep(0.5)  # Wait before retry
    return None  # Not found
```

**Savings:**
```
Without expanding ring:
- Always TTL=3: 910 messages
- 100 queries: 91,000 messages

With expanding ring:
- 80% found at TTL=1: 80 × 10 = 800
- 15% found at TTL=2: 15 × 100 = 1,500
- 5% found at TTL=3: 5 × 910 = 4,550
- Total: 6,850 messages ✅

Savings: 91,000 - 6,850 = 84,150 (92% reduction!) 🎉
```

**2. Random Walk (Alternative approach)**
```
Instead of flooding to ALL neighbors:
→ Forward to ONE random neighbor

Algorithm:
1. Pick random neighbor
2. Forward query (TTL--)
3. If not found, try another random walk

Comparison:
─────────────────────────────────────────
              Flooding   Random Walk
Messages:        910          100
Coverage:       9.1%          1%
Success (r=1%): 99.9%        63%

For rare files (r=0.1%):
Multiple random walks (5×):
- Messages: 5 × 100 = 500 ✅
- Coverage: 5%
- Success: 95% ✅
- Still 2× better than flooding!
```

**3. Bloom Filter-based Duplicate Detection**
```python
from bloom_filter import BloomFilter

class Node:
    def __init__(self):
        # Bloom filter for seen query IDs
        self.bloom = BloomFilter(max_elements=10000, error_rate=0.01)
    
    def search(self, query_id, keyword, ttl):
        # Check if already seen (fast!)
        if query_id in self.bloom:
            return  # Likely duplicate, drop
        
        # Add to bloom filter
        self.bloom.add(query_id)
        
        # Process query...
        if ttl > 0:
            for neighbor in self.neighbors:
                neighbor.search(query_id, keyword, ttl-1)

Benefits:
✅ Space: 12 KB vs 400 KB (hash set)
✅ Speed: O(1) lookup
✅ Reduces duplicate messages by ~70%
⚠️ Small false positive rate (1%)
```

**4. Super-Peer Architecture**
```
Hybrid P2P: Combine flooding with centralized index

┌────────────────────────────────────────┐
│         SUPER PEERS (High capacity)    │
│  ┌──────┐  ┌──────┐  ┌──────┐         │
│  │ SP1  │──│ SP2  │──│ SP3  │         │
│  └──┬───┘  └──┬───┘  └──┬───┘         │
└─────┼─────────┼─────────┼──────────────┘
      │         │         │
   ┌──┴──┐   ┌─┴──┐   ┌──┴──┐
   │ P1  │   │ P2 │   │ P3  │  Regular peers
   │ P4  │   │ P5 │   │ P6  │
   │ P7  │   │ P8 │   │ P9  │
   └─────┘   └────┘   └─────┘

Search process:
1. Query sent to super-peer
2. Super-peer floods to other super-peers (small network)
3. Each super-peer checks its local peers

Messages:
- Flooding among 100 super-peers: ~1,000 messages
- Local search: ~10 messages per SP
- Total: ~2,000 messages ✅
- vs 91,000 in pure flooding! (95% reduction 🎉)
```

**5. TTL Adaptation based on Query Type**
```python
def adaptive_ttl(query_type, keyword):
    """
    Adjust TTL based on expected replication
    """
    if query_type == "popular":
        # Popular files (music, movies)
        return 1  # Found quickly ✅
    
    elif query_type == "common":
        # Moderately replicated
        return 2
    
    elif query_type == "rare":
        # Rare or specific files
        return 5  # Need wider search
    
    elif query_type == "keyword_search":
        # Broad search
        return 3
    
    else:
        return 3  # Default

# Machine learning approach
def predict_ttl(keyword, history):
    """
    Learn optimal TTL from past queries
    """
    # Features: keyword length, frequency, past success
    # Use random forest or neural network
    # Output: Predicted optimal TTL
    pass
```

---

## 📊 Tóm tắt

### Key Points

- ✅ **Flooding with TTL=3**: 911 nodes, 910 messages
- ✅ **Coverage**: 9.11% of 10K-node network
- ✅ **Success rate**: >99% for popular files (r≥1%)
- ⚠️ **Overhead**: Exponential growth (9x per TTL)
- ❌ **Rare files**: Poor success rate (9% for r=0.01%)

### Trade-offs
```
╔═══════════════════════════════════════════════╗
║            FLOODING TRADE-OFFS                ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Increase TTL:                                ║
║  ✅ Higher coverage                           ║
║  ✅ Better for rare files                     ║
║  ❌ Exponentially more messages               ║
║  ❌ Network congestion                        ║
║                                               ║
║  Decrease TTL:                                ║
║  ✅ Lower overhead                            ║
║  ✅ Less congestion                           ║
║  ❌ Lower coverage                            ║
║  ❌ Miss rare files                           ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

### Khuyến nghị

**Cho mạng k=10, N=10K:**

| File Type | Strategy | TTL | Messages | Success |
|-----------|----------|-----|----------|---------|
| Popular (r>1%) | Expanding ring | 1→2→3 | ~100 ✅ | 99%+ |
| Common (r~0.1%) | Fixed TTL | 3 | 910 ⚠️ | 60% |
| Rare (r<0.01%) | Random walk × 5 | 100 each | 500 ✅ | 95% |
| Critical | Super-peer | 2 (SPs) | ~2,000 | 99% |

---

## 🔗 Tài liệu tham khảo

### Papers
- **"A Measurement Study of Peer-to-Peer File Sharing Systems"** - Saroiu et al., 2002
- **"Search in Power-Law Networks"** - Adamic et al., 2001

### Systems
- **Gnutella**: Early P2P using flooding
- **Kazaa**: Super-peer architecture
- **BitTorrent**: DHT-based (no flooding)

---

## 🧭 Navigation

**[⬅️ Câu 3: Chord DHT](./cau-3-chord-dht.md)** | **[📚 Quay lại Chương 2](./README.md)** | **[➡️ Câu 5: Random Walk](./cau-5-random-walk.md)**

---

*Cập nhật lần cuối: 11/12/2025*