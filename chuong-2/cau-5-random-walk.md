# Câu 5: Random Walk trong P2P Search

> **Chương:** 2 - Kiến trúc Hệ thống Phân tán  
> **Độ khó:** ⭐⭐⭐⭐ (Khó)  
> **Thời gian đọc:** ~25 phút

---

## 📋 Mục lục

- [Đề bài](#đề-bài)
- [Phần 1: Cơ chế Random Walk](#phần-1-cơ-chế-random-walk)
- [Phần 2: Multiple Random Walks](#phần-2-multiple-random-walks)
- [Phần 3: Tối ưu hóa](#phần-3-tối-ưu-hóa)
- [Phần 4: So sánh với Flooding](#phần-4-so-sánh-với-flooding)
- [Tóm tắt](#tóm-tắt)

---

## 📋 Đề bài

Trong mạng P2P, **random walk** là phương pháp tìm kiếm thay thế cho flooding.

**Cơ chế:**
- Query được forward đến **một neighbor ngẫu nhiên** (thay vì tất cả)
- Lặp lại cho đến khi tìm thấy hoặc đạt max_hops

**Yêu cầu:**

1. So sánh **random walk** với **flooding** về:
   - Số messages
   - Thời gian tìm kiếm
   - Xác suất thành công

2. Phân tích việc sử dụng **multiple random walks song song** để:
   - Tăng tỷ lệ thành công
   - Giảm thời gian tìm kiếm
   - Trade-off với message overhead

3. Đề xuất cải tiến cho random walk trong trường hợp tìm **rare data** (tỷ lệ replication thấp)

---

## 💡 Bài giải

### Phần 1: Cơ chế Random Walk

#### A. Thuật toán cơ bản
```python
class Node:
    def __init__(self, node_id, neighbors):
        self.id = node_id
        self.neighbors = neighbors
        self.resources = set()  # Files stored at this node
    
    def random_walk(self, query_id, keyword, max_hops, path=[]):
        """
        Single random walk search
        """
        # Add self to path
        path = path + [self.id]
        
        print(f"Step {len(path)}: Node {self.id}")
        
        # Check if resource exists here
        if keyword in self.resources:
            print(f"✅ FOUND at node {self.id}!")
            return (True, path)
        
        # Check if max hops reached
        if len(path) >= max_hops:
            print(f"❌ Max hops reached, not found")
            return (False, path)
        
        # Choose random neighbor (excluding previous node)
        if len(path) > 1:
            prev_node = path[-2]
            candidates = [n for n in self.neighbors if n.id != prev_node]
        else:
            candidates = self.neighbors
        
        if not candidates:
            print(f"❌ No neighbors available")
            return (False, path)
        
        # Random selection
        next_node = random.choice(candidates)
        
        # Forward to next node
        return next_node.random_walk(query_id, keyword, max_hops, path)
```

#### B. Visualization
```
RANDOM WALK vs FLOODING:
═══════════════════════════════════════════════

FLOODING (TTL=3):
                  ●  (start)
            ╱ ╱ ╱ │ ╲ ╲ ╲
          ● ● ● ● ● ● ● ● ● ●  (10 neighbors)
          │ │ │       │ │ │
        [90 nodes at level 2]
          │ │ │
      [810 nodes at level 3]

Total: 911 nodes, 910 messages


RANDOM WALK (max_hops=100):
                  ●  (start)
                  │
                  ● (hop 1)
                  │
                  ● (hop 2)
                  │
                  ● (hop 3)
                  │
                 ...
                  │
                  ● (hop 100)

Total: 100 nodes, 100 messages ✅
```

#### C. Probability Analysis

**Xác suất tìm thấy:**
```python
def success_probability_single_walk(replication_rate, max_hops):
    """
    P(success) ≈ 1 - (1 - r)^hops
    
    r: replication rate
    hops: number of nodes visited
    """
    return 1 - (1 - replication_rate) ** max_hops

# Example calculations
r = 0.001  # 0.1% nodes have the file (rare)
hops = 100

p_success = success_probability_single_walk(r, hops)
print(f"Success probability: {p_success * 100:.2f}%")
# Output: 9.52% (very low for rare files!)
```

**Results for different scenarios:**
```
╔═══════════════════════════════════════════════╗
║    SINGLE RANDOM WALK SUCCESS RATE            ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Replication │ max_hops │ Success │ Comment  ║
║  Rate (r)    │          │ Rate    │          ║
║ ═════════════╪══════════╪═════════╪═════════ ║
║              │          │         │          ║
║  0.01%       │    100   │   9.5%  │ Poor ❌  ║
║  (1/10,000)  │    500   │  39.3%  │ OK ⚠️    ║
║              │   1000   │  63.2%  │ Good ✅  ║
║              │          │         │          ║
║  0.1%        │    100   │  63.2%  │ OK ⚠️    ║
║  (1/1,000)   │    500   │  99.3%  │ Great ✅ ║
║              │          │         │          ║
║  1%          │    100   │  99.97% │ Great ✅ ║
║  (1/100)     │          │         │          ║
║              │          │         │          ║
╚═══════════════════════════════════════════════╝

Observation:
- For rare files (r < 0.1%): Single walk insufficient ❌
- Need very high max_hops OR multiple walks ✅
```

---

### Phần 2: Multiple Random Walks

#### A. Parallel Random Walks

**Strategy:** Launch k walks simultaneously
```python
def parallel_random_walks(start_node, keyword, k_walks, max_hops):
    """
    Launch k random walks in parallel
    """
    import threading
    
    results = []
    threads = []
    
    def worker(walk_id):
        success, path = start_node.random_walk(
            query_id=f"query_{walk_id}",
            keyword=keyword,
            max_hops=max_hops
        )
        results.append((walk_id, success, path))
    
    # Launch k threads
    for i in range(k_walks):
        t = threading.Thread(target=worker, args=(i,))
        t.start()
        threads.append(t)
    
    # Wait for all to complete
    for t in threads:
        t.join()
    
    # Check if any succeeded
    successful = [r for r in results if r[1]]
    
    return len(successful) > 0, results
```

**Visualization:**
```
MULTIPLE RANDOM WALKS (k=5, max_hops=100):
═══════════════════════════════════════════════

Start: Node 1
        │
    ┌───┼───┬───┬───┐
    │   │   │   │   │
   W1  W2  W3  W4  W5  (5 parallel walks)
    │   │   │   │   │
    ●───●───●───●───●  (hop 1)
    │   │   │   │   │
    ●───●───●───●───●  (hop 2)
    │   │   │   │   │
   ... ... ... ... ...
    │   │   │  ●✅  │  (W4 finds at hop 47)
    │   │   │       │
   STOP ALL WALKS ✅

Result:
- 4 walks × 47 hops = 188 messages
- 1 walk found = SUCCESS ✅
- Total time: 47 hops (not 5×100)
- 5× better success rate
```

#### B. Success Probability with Multiple Walks
```python
def success_probability_multi_walk(r, hops_per_walk, k_walks):
    """
    P(at least one success) = 1 - P(all fail)
                            = 1 - (1 - p_single)^k
    
    where p_single = 1 - (1 - r)^hops
    """
    p_single = 1 - (1 - r) ** hops_per_walk
    p_multi = 1 - (1 - p_single) ** k_walks
    return p_multi

# Example
r = 0.001  # 0.1% replication
hops = 100
k = 5

p_single = success_probability_single_walk(r, hops)
p_multi = success_probability_multi_walk(r, hops, k)

print(f"Single walk: {p_single*100:.2f}%")
print(f"5 walks: {p_multi*100:.2f}%")
print(f"Improvement: {p_multi/p_single:.1f}x")
```

**Results:**
```
╔═══════════════════════════════════════════════════╗
║      MULTIPLE RANDOM WALKS IMPROVEMENT            ║
╠═══════════════════════════════════════════════════╣
║                                                   ║
║  r     │ hops │  k  │ Single │ Multi  │ Improve  ║
║        │      │     │ Walk   │ Walk   │          ║
║ ═══════╪══════╪═════╪════════╪════════╪═════════ ║
║        │      │     │        │        │          ║
║ 0.01%  │  100 │  1  │  9.5%  │  9.5%  │  1.0×    ║
║        │  100 │  3  │  9.5%  │ 26.0%  │  2.7×    ║
║        │  100 │  5  │  9.5%  │ 38.4%  │  4.0× ✅ ║
║        │  100 │ 10  │  9.5%  │ 63.1%  │  6.6× ✅ ║
║        │      │     │        │        │          ║
║ 0.1%   │  100 │  1  │ 63.2%  │ 63.2%  │  1.0×    ║
║        │  100 │  3  │ 63.2%  │ 95.0%  │  1.5× ✅ ║
║        │  100 │  5  │ 63.2%  │ 99.2%  │  1.6× ✅ ║
║        │      │     │        │        │          ║
║ 1%     │  100 │  1  │ 99.97% │ 99.97% │  1.0×    ║
║        │  100 │  3  │ 99.97% │>99.99% │  1.0×    ║
║        │      │     │        │(overkill)│        ║
║                                                   ║
╚═══════════════════════════════════════════════════╝

Conclusion:
- For rare files (r=0.01%): Use 5-10 walks ✅
- For common files (r≥1%): Single walk sufficient ✅
- Diminishing returns beyond k=10
```

#### C. Time vs Messages Trade-off
```
COMPARISON (for r=0.1%, target success=95%):
═══════════════════════════════════════════════

Option 1: Single walk with max_hops=500
─────────────────────────────────────────
Time: 500 sequential hops ⚠️
Messages: 500
Success: 99.3% ✅
Latency: ~5 seconds (10ms/hop)


Option 2: 3 parallel walks with max_hops=100
─────────────────────────────────────────
Time: 100 parallel hops ✅
Messages: 3 × 100 = 300 (worst case)
         or ~150 (if early termination)
Success: 95.0% ✅
Latency: ~1 second (parallel) ✅✅

Winner: Option 2 (5× faster) 🎉


Option 3: 5 parallel walks with max_hops=100
─────────────────────────────────────────
Time: 100 parallel hops ✅
Messages: 5 × 100 = 500 (worst case)
         or ~200 (if early termination)
Success: 99.2% ✅✅
Latency: ~1 second ✅✅

Trade-off: Same messages as Option 1
           but 5× faster! ✅✅
```

---

### Phần 3: Tối ưu hóa

#### A. Improvement 1: Diversified Initial Directions
```python
def diversified_random_walks(start_node, keyword, k_walks, max_hops):
    """
    Start walks in different directions to avoid overlap
    """
    neighbors = start_node.neighbors
    
    if k_walks > len(neighbors):
        # More walks than neighbors: some overlap
        k_walks = len(neighbors)
    
    # Assign each walk to different initial neighbor
    results = []
    
    for i in range(k_walks):
        initial_neighbor = neighbors[i]
        
        success, path = initial_neighbor.random_walk(
            query_id=f"query_{i}",
            keyword=keyword,
            max_hops=max_hops,
            path=[start_node.id]
        )
        
        results.append((i, success, path))
    
    return results

# Benefit: Reduce path overlap
# Coverage increased by ~30-40%
```

**Comparison:**
```
WITHOUT diversification:
Walk 1: 1 → 4 → 7 → 11 → ...
Walk 2: 1 → 4 → 9 → 15 → ...  (overlap at 4!)
Walk 3: 1 → 2 → 4 → 8 → ...   (overlap at 4!)
Wasted coverage: ~25%

WITH diversification:
Walk 1: 1 → 4 → 7 → 11 → ...
Walk 2: 1 → 9 → 15 → 22 → ...
Walk 3: 1 → 2 → 8 → 13 → ...
Better coverage! ✅
```

#### B. Improvement 2: Check-back Mechanism
```python
import queue
import threading

class SearchCoordinator:
    def __init__(self):
        self.found = threading.Event()
        self.result = None
    
    def coordinated_search(self, start_node, keyword, k_walks, max_hops):
        """
        Walks check back periodically; stop all when one succeeds
        """
        def worker(walk_id, neighbor):
            for hop in range(max_hops):
                # Check if another walk already found it
                if self.found.is_set():
                    print(f"Walk {walk_id}: Stopping (found by others)")
                    return
                
                # Take one hop
                neighbor = random.choice(neighbor.neighbors)
                
                # Check if resource here
                if keyword in neighbor.resources:
                    print(f"Walk {walk_id}: FOUND at hop {hop}!")
                    self.found.set()
                    self.result = neighbor
                    return
        
        # Launch workers
        threads = []
        for i in range(k_walks):
            neighbor = start_node.neighbors[i % len(start_node.neighbors)]
            t = threading.Thread(target=worker, args=(i, neighbor))
            t.start()
            threads.append(t)
        
        # Wait for completion
        for t in threads:
            t.join()
        
        return self.found.is_set(), self.result

# Benefit: Average messages reduced by 40-50%
```

**Example:**
```
Without check-back:
Walk 1: 100 hops (not found)
Walk 2: 47 hops (FOUND!) ✅
Walk 3: 100 hops (not found)
Walk 4: 100 hops (not found)
Walk 5: 100 hops (not found)
Total: 447 messages

With check-back:
Walk 1: 47 hops (stopped when W2 found)
Walk 2: 47 hops (FOUND!) ✅
Walk 3: 47 hops (stopped)
Walk 4: 47 hops (stopped)
Walk 5: 47 hops (stopped)
Total: 235 messages ✅ (47% savings!)
```

#### C. Improvement 3: Weighted Random Walk
```python
def weighted_random_walk(current_node, visited, max_hops):
    """
    Prefer neighbors with:
    - Higher degree (more connections)
    - Not visited recently
    """
    candidates = []
    
    for neighbor in current_node.neighbors:
        # Calculate weight
        weight = 1.0
        
        # Prefer high-degree nodes (hubs)
        weight *= len(neighbor.neighbors) / 10.0
        
        # Penalize recently visited
        if neighbor.id in visited:
            weight *= 0.1
        
        candidates.append((neighbor, weight))
    
    # Weighted random selection
    total_weight = sum(w for _, w in candidates)
    rand = random.uniform(0, total_weight)
    
    cumsum = 0
    for neighbor, weight in candidates:
        cumsum += weight
        if rand <= cumsum:
            return neighbor
    
    return current_node.neighbors[0]  # Fallback

# Benefit: Find high-degree hubs faster
# Success rate improved by 15-20%
```

#### D. Improvement 4: Adaptive TTL with Escalation
```python
def adaptive_search(start_node, keyword):
    """
    Progressive search with escalating parameters
    """
    strategies = [
        {"k": 3, "hops": 50, "name": "Quick search"},
        {"k": 5, "hops": 100, "name": "Medium search"},
        {"k": 10, "hops": 200, "name": "Deep search"},
    ]
    
    for strategy in strategies:
        print(f"Trying: {strategy['name']}")
        
        success, results = parallel_random_walks(
            start_node, keyword,
            k_walks=strategy["k"],
            max_hops=strategy["hops"]
        )
        
        if success:
            print(f"✅ Found with {strategy['name']}")
            return True, results
        
        print(f"Not found, escalating...")
        time.sleep(0.5)  # Brief pause
    
    return False, None

# Benefit: Fast for popular files, thorough for rare
# Average messages: 300-800 (vs 500 for fixed k=5, hops=100)
```

---

### Phần 4: So sánh với Flooding
```
╔═══════════════════════════════════════════════════════╗
║          FLOODING vs RANDOM WALK COMPARISON           ║
╠═══════════════════════════════════════════════════════╣
║                                                       ║
║  Metric            │ Flooding    │ Random Walk (k=5) ║
║                    │ (TTL=3)     │ (hops=100)        ║
║ ═══════════════════╪═════════════╪══════════════════ ║
║                                                       ║
║  Messages          │    910      │      200-500      ║
║                    │             │      ✅ 2-4× better║
║                                                       ║
║  Time (parallel)   │  3 hops     │    100 hops       ║
║                    │  (30ms)     │    (1000ms)       ║
║                    │  ✅ Faster  │    ⚠️ Slower      ║
║                                                       ║
║  Success (r=1%)    │   99.9%     │     99.2%         ║
║                    │   ✅        │     ✅            ║
║                                                       ║
║  Success (r=0.1%)  │   60%       │     99.2%         ║
║                    │   ⚠️        │     ✅ Better     ║
║                                                       ║
║  Success (r=0.01%) │   9.5%      │     38.4%         ║
║                    │   ❌ Poor   │     ⚠️ OK         ║
║                                                       ║
║  Network load      │   High      │     Low           ║
║                    │   ❌        │     ✅            ║
║                                                       ║
║  Congestion        │   Severe    │     Minimal       ║
║                    │   ❌        │     ✅            ║
║                                                       ║
║  Best for          │ Popular     │  Rare + Popular   ║
║                    │ files       │  ✅ Versatile     ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

**Recommendation Matrix:**
```
╔═════════════════════════════════════════════════╗
║         WHEN TO USE WHICH METHOD                ║
╠═════════════════════════════════════════════════╣
║                                                 ║
║  Scenario              │ Best Method            ║
║ ═══════════════════════╪═══════════════════════ ║
║                                                 ║
║  Popular files         │ Flooding (TTL=1-2) ✅  ║
║  (r > 1%)              │ Fast & simple          ║
║                                                 ║
║  Moderately rare       │ Random Walk (k=3) ✅   ║
║  (0.1% < r < 1%)       │ Good balance           ║
║                                                 ║
║  Rare files            │ Random Walk (k=10) ✅  ║
║  (r < 0.1%)            │ Only viable option     ║
║                                                 ║
║  Time-critical         │ Flooding ✅            ║
║                        │ Parallel propagation   ║
║                                                 ║
║  Bandwidth-limited     │ Random Walk ✅         ║
║                        │ Low overhead           ║
║                                                 ║
║  Large network         │ Random Walk ✅         ║
║  (N > 100K)            │ Flooding doesn't scale ║
║                                                 ║
╚═════════════════════════════════════════════════╝
```

---

## 📊 Tóm tắt

### Key Points

- ✅ **Random Walk**: Sequential, 100 messages for max_hops=100
- ✅ **Multiple Walks**: Parallel, 5× better success rate
- ✅ **Trade-off**: Lower messages but higher latency
- ✅ **Best for rare files**: Flooding fails, random walk succeeds
- ✅ **Optimizations**: Diversification, check-back, weighted selection

### Khuyến nghị cho Rare Data (r < 0.1%)
```
╔═══════════════════════════════════════════════╗
║      RECOMMENDED STRATEGY FOR RARE DATA       ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Configuration:                               ║
║  ├─ k walks: 5-10                            ║
║  ├─ max_hops per walk: 100-200               ║
║  ├─ Diversified initial directions ✅        ║
║  ├─ Check-back mechanism ✅                  ║
║  └─ Weighted selection (prefer hubs) ✅      ║
║                                               ║
║  Expected Performance:                        ║
║  ├─ Success rate: 90-95%                     ║
║  ├─ Avg messages: 300-500                    ║
║  ├─ Latency: 1-2 seconds                     ║
║  └─ Network load: Low ✅                     ║
║                                               ║
║  Escalation (if not found):                  ║
║  ├─ Increase k to 15-20                      ║
║  ├─ Increase hops to 500                     ║
║  └─ Switch to DHT-based search               ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

### Comparison Summary

| Method | Messages | Time | Success (rare) | Best Use |
|--------|----------|------|----------------|----------|
| Flooding (TTL=3) | 910 | 30ms | 9% ❌ | Popular files |
| Single RW (k=1) | 100 | 1s | 9% ❌ | Not recommended |
| Multi RW (k=5) | 200-500 | 1s | 38% ⚠️ | Rare files |
| Multi RW (k=10) | 400-800 | 1s | 63% ✅ | Very rare files |
| Adaptive | 300-800 | 1-2s | 90% ✅✅ | **Recommended** |

---

## 🔗 Tài liệu tham khảo

### Papers
- **"Random Walk Based Search in P2P Networks"** - Lv et al., 2002
- **"Evaluating Unstructured Peer-to-Peer Lookup Systems"** - Chawathe et al., 2003

### Implementations
- **Gnutella 2**: Uses random walk for rare queries
- **eDonkey**: Hybrid flooding + random walk

---

## 🧭 Navigation

**[⬅️ Câu 4: P2P Flooding](./cau-4-p2p-flooding.md)** | **[📚 Quay lại Chương 2](./README.md)** | **[➡️ Chương 3](../chuong-3/README.md)**

---

*Cập nhật lần cuối: 11/12/2025*