CHƯƠNG 2 - CÂU 5
Đề bài:
Trong một mạng P2P không cấu trúc, một nút cần tìm kiếm dữ liệu hiếm (rare data). Hãy áp dụng phương pháp random walk để mô tả cách tìm kiếm dữ liệu này. Đề xuất và giải thích cách cải tiến (ví dụ: khởi động nhiều random walks đồng thời) để giảm thời gian tìm thấy dữ liệu, và phân tích sự đánh đổi giữa thời gian tìm kiếm và lưu lượng mạng.

BÀI GIẢI:
Phần 1: Kiến thức nền tảng về Random Walk
A. Random Walk là gì?
Định nghĩa:
Random Walk là thuật toán tìm kiếm trong mạng P2P không cấu trúc, trong đó query message "đi bộ ngẫu nhiên" từ node này sang node khác, thay vì broadcast như Flooding.
Đặc điểm:

Mỗi bước chọn 1 neighbor ngẫu nhiên để forward
Tiếp tục cho đến khi tìm thấy hoặc đạt giới hạn (max hops)
Giảm drastically số messages so với flooding

So sánh với Flooding:
FLOODING (TTL=3):                 RANDOM WALK (max_hops=10):
       [S]                                [S]
      / | \                                |
    [A][B][C]                            [A]
   /|\ |\ |\                              |
  [...100+ nodes...]                    [D]
                                          |
Messages: ~1000+                        [H] ← Found!
                                          
                                    Messages: 10

B. Tại sao Random Walk phù hợp với rare data?
Vấn đề với rare data:
Rare data = Dữ liệu có replication ratio thấp
Ví dụ: File chỉ có ở 0.01% nodes trong mạng

Với Flooding (TTL=3):
- Coverage: 8% mạng (800/10,000 nodes)
- Probability tìm thấy: 1 - (1-0.0001)^800 ≈ 7.7%
- Chi phí: 1,110 messages
→ Vừa tốn kém vừa không hiệu quả!
Random Walk advantages:
✅ Chi phí thấp: O(k) messages với k = max_hops
✅ Có thể kéo dài để tăng coverage (k = 100, 1000...)
✅ Không gây congestion
✅ Thích hợp cho rare data cần tìm kiếm sâu

Phần 2: Mô tả chi tiết Random Walk cơ bản
Giả định:
- Mạng P2P: 10,000 nodes
- Rare file: "rare_book.pdf" có ở 10 nodes (0.1% replication)
- Node S (Source) cần tìm file
- Max hops: 100
Topology mạng:
    [S] ──── [A] ──── [D] ──── [G]
     |        |        |        |
    [B] ──── [C] ──── [E] ──── [H]
     |        |        |        |
    [...]──[...]──[...]──[Target🎯]

THUẬT TOÁN RANDOM WALK CƠ BẢN:
pythonclass RandomWalkSearch:
    def __init__(self, max_hops=100):
        self.max_hops = max_hops
        self.visited_nodes = set()
    
    def search(self, filename, source_node):
        """
        Thực hiện random walk search
        """
        current_node = source_node
        hops = 0
        path = [source_node]
        
        while hops < self.max_hops:
            # Bước 1: Kiểm tra node hiện tại
            if current_node.has_file(filename):
                return {
                    'found': True,
                    'node': current_node,
                    'hops': hops,
                    'path': path
                }
            
            # Bước 2: Đánh dấu đã visit (tránh loop)
            self.visited_nodes.add(current_node.id)
            
            # Bước 3: Chọn neighbor ngẫu nhiên
            neighbors = current_node.get_neighbors()
            
            # Filter: loại bỏ nodes đã visit (optional)
            unvisited = [n for n in neighbors 
                        if n.id not in self.visited_nodes]
            
            if not unvisited:
                # Deadend: Tất cả neighbors đã visit
                # Option 1: Backtrack
                # Option 2: Chọn random từ tất cả neighbors
                candidates = neighbors
            else:
                candidates = unvisited
            
            # Bước 4: Random selection
            next_node = random.choice(candidates)
            
            # Bước 5: Di chuyển đến node tiếp theo
            current_node = next_node
            path.append(current_node)
            hops += 1
        
        # Không tìm thấy sau max_hops
        return {
            'found': False,
            'hops': hops,
            'path': path
        }
```

---

**CHI TIẾT CÁC BƯỚC:**

**Bước khởi đầu (Hop 0):**
```
Current Node: S
Action: Check local storage
Result: File không có
Neighbors: [A, B]
Random selection: A (50% probability)
```

**Hop 1:**
```
Current Node: A
Visited: {S}
Check: File không có
Neighbors: [S, C, D] 
Unvisited neighbors: [C, D] (loại S)
Random selection: D (50% probability)
```

**Hop 2:**
```
Current Node: D
Visited: {S, A}
Check: File không có
Neighbors: [A, E, G]
Unvisited: [E, G]
Random selection: E (50% probability)
```

**Hop 3-99:**
```
Continue random walk...
Node path: S → A → D → E → H → K → ... (random)
Each hop: 1 message
Total messages so far: 99
```

**Hop 100:**
```
Current Node: X
Check: File CÓ ở đây! 🎯
Return: {
    found: True,
    node: X,
    hops: 100,
    path: [S, A, D, E, ..., X]
}
```

---

**Ví dụ path cụ thể:**
```
Random Walk Path (100 hops):
S → A → D → E → H → K → L → P → Q → R
  → T → W → X → Y → Z → M → N → O → S (back to S!)
  → B → C → F → I → J → ... → Target🎯

Visualization:
    Start
      ↓
    [S]────[A]────[D]
     ↓      ↓      ↓
    [B]────[C]────[E]
            ↓      ↓
          [F]    [H]────[K]
                  ↓      ↓
                [J]    [L]────→ ... → [Target🎯]
```

**Thống kê:**
```
Total hops: 100
Messages sent: 100
Nodes visited: 100 (hoặc ít hơn nếu revisit)
Success: YES (tìm thấy)
```

---

**Phân tích hiệu quả:**

**Coverage của Random Walk:**
```
Với max_hops = k:
- Best case: Visit k unique nodes
- Worst case: Visit < k nodes (do loops)
- Expected: ~0.7k unique nodes (với loops)

So với Flooding (TTL=3):
- Flooding: 800 nodes, 1,110 messages
- Random Walk (k=100): 70-100 nodes, 100 messages

→ Random Walk: Ít messages hơn 11x
→ Nhưng coverage thấp hơn 8x
```

**Probability tìm thấy rare data:**
```
Giả sử replication ratio r = 0.001 (0.1%)
Total nodes N = 10,000
Nodes có file = 10

Với max_hops = k:
Prob(found) ≈ 1 - (1 - r)^k

k = 100:  P ≈ 9.5%
k = 500:  P ≈ 39%
k = 1000: P ≈ 63%
k = 5000: P ≈ 99.3%

→ Cần max_hops rất lớn cho rare data!

Phần 3: Cải tiến - Multiple Random Walks
A. Parallel Random Walks (Đi bộ song song)
Ý tưởng:
Thay vì 1 walker, khởi động nhiều walkers đồng thời từ source node.
Thuật toán:
pythonclass MultipleRandomWalks:
    def __init__(self, num_walkers=5, max_hops_per_walker=200):
        self.num_walkers = num_walkers
        self.max_hops_per_walker = max_hops_per_walker
    
    def search(self, filename, source_node):
        """
        Khởi động nhiều random walks song song
        """
        walkers = []
        
        # Khởi tạo multiple walkers
        for i in range(self.num_walkers):
            walker = RandomWalker(
                walker_id=i,
                max_hops=self.max_hops_per_walker
            )
            walkers.append(walker)
        
        # Execute parallel walks
        results = parallel_execute([
            walker.walk(filename, source_node)
            for walker in walkers
        ])
        
        # Aggregate results
        for result in results:
            if result['found']:
                return result  # Trả về walker đầu tiên tìm thấy
        
        return {'found': False}
```

**Visualization:**
```
                Source Node [S]
                     |
        ┌────────────┼────────────┐
        |            |            |
    Walker 1     Walker 2     Walker 3
        |            |            |
       [A]         [B]          [C]
        |            |            |
       [D]         [E]          [F]
        |            |            |
       [G]       [Target🎯]     [H]
        |                         |
       ...                       ...

Walker 2 finds target at hop 3!
Other walkers terminate immediately.
```

---

**BƯỚC CHI TIẾT:**

**Initialization (t=0):**
```
Source: Node S
Launch 5 walkers: W1, W2, W3, W4, W5

W1 → neighbor A
W2 → neighbor B
W3 → neighbor B (có thể trùng)
W4 → neighbor A
W5 → neighbor C
```

**Round 1 (each walker hops once):**
```
W1: S → A → D     (2 hops, 2 messages)
W2: S → B → E     (2 hops, 2 messages)
W3: S → B → F     (2 hops, 2 messages)
W4: S → A → G     (2 hops, 2 messages)
W5: S → C → H     (2 hops, 2 messages)

Total: 10 messages
```

**Round 2:**
```
W1: ... → D → I
W2: ... → E → Target🎯 (FOUND!)

→ Broadcast termination signal to all walkers
→ W1, W3, W4, W5 stop immediately
```

**Result:**
```
Success: YES
Winner: Walker 2
Total hops: 3 (S → B → E → Target)
Total messages: ~15 (all walkers combined before termination)
Time: 3 rounds

B. Cải tiến: Adaptive Walker Strategy
1. Diversified initial directions:
pythondef launch_diverse_walkers(source_node, num_walkers):
    """
    Đảm bảo walkers đi các hướng khác nhau
    """
    neighbors = source_node.get_neighbors()
    
    # Assign mỗi walker một initial neighbor khác nhau
    for i in range(num_walkers):
        initial_neighbor = neighbors[i % len(neighbors)]
        walker = Walker(initial_direction=initial_neighbor)
        launch(walker)
```

**Lợi ích:**
```
Tránh tình huống: 5 walkers cùng đi 1 hướng
→ Tăng coverage area
→ Tăng success probability

2. Check-back mechanism:
pythonclass SmartWalker:
    def walk(self):
        for hop in range(self.max_hops):
            # Random walk
            current_node = self.random_step()
            
            # Định kỳ check với source
            if hop % 10 == 0:
                if self.check_if_others_found():
                    return  # Dừng nếu walker khác đã tìm thấy
```

**Lợi ích:**
```
Tránh lãng phí: Walkers không tìm kiếm tiếp sau khi đã có kết quả
→ Giảm overhead messages

3. TTL for walkers:
pythonclass TTLWalker:
    def __init__(self, walker_id, initial_ttl=50):
        self.walker_id = walker_id
        self.ttl = initial_ttl
    
    def walk(self):
        while self.ttl > 0:
            current_node = self.random_step()
            
            if current_node.has_file(target):
                return SUCCESS
            
            self.ttl -= 1  # Decrement
        
        return TIMEOUT
```

**Lợi ích:**
```
Giới hạn mỗi walker, tránh "đi mãi không về"
→ Bounded execution time
→ Predictable resource usage

4. Weighted random walk:
pythondef weighted_random_selection(neighbors):
    """
    Chọn neighbor dựa trên metadata (thay vì uniform random)
    """
    weights = []
    for neighbor in neighbors:
        weight = calculate_weight(neighbor)
        weights.append(weight)
    
    # Weighted random choice
    return random.choices(neighbors, weights=weights)[0]

def calculate_weight(node):
    """
    Heuristics để ưu tiên nodes có khả năng cao hơn
    """
    score = 0
    
    # Ưu tiên nodes có nhiều files
    score += node.num_files * 0.5
    
    # Ưu tiên nodes có degree cao (hub)
    score += len(node.neighbors) * 0.3
    
    # Ưu tiên nodes ít được visit
    if node.id not in visited:
        score += 10
    
    return score
```

**Lợi ích:**
```
Thông minh hơn uniform random
→ Tăng probability tìm thấy
→ Giảm expected hops
```

---

#### **Phần 4: Phân tích Trade-off**

**A. So sánh các phương pháp**

**Bảng so sánh:**

| Method | Messages | Time (hops) | Success Rate | Congestion |
|--------|----------|-------------|--------------|------------|
| **Flooding (TTL=3)** | 1,110 | 3 | 7.7% (rare) | ❌ High |
| **Single Random Walk (k=100)** | 100 | 100 | 9.5% (rare) | ✅ Low |
| **Multiple Walks (5 walkers, k=100)** | 500 | ~20 | 39% (rare) | ⚠️ Medium |
| **Adaptive Walks (5 walkers)** | 300-400 | ~15 | 45% (rare) | ⚠️ Medium |

---

**B. Trade-off chi tiết: Messages vs Time**

**Scenario 1: Single Random Walk**
```
Configuration:
- 1 walker
- max_hops = 1000

Pros:
✅ Minimal messages: 1,000
✅ No congestion
✅ Simple implementation

Cons:
❌ Slow: 1,000 sequential hops
❌ Low success rate: ~63% for rare data
❌ High latency: 1,000 × RTT

Use case: 
- Non-urgent queries
- Bandwidth-constrained networks
- Low-priority searches
```

**Example timeline:**
```
Time: 0s    → Start
Time: 1s    → Hop 100
Time: 10s   → Hop 1000 (Found or Timeout)

Total time: 10 seconds (assuming 10ms per hop)
Total messages: 1,000
```

---

**Scenario 2: Multiple Random Walks (5 walkers)**
```
Configuration:
- 5 walkers
- max_hops per walker = 200

Pros:
✅ Faster: Parallel execution
✅ Higher success rate: ~92% for rare data
✅ Better coverage

Cons:
❌ More messages: 5 × 200 = 1,000 (worst case)
⚠️ Medium congestion (5x single walker)
❌ More complex coordination

Use case:
- Time-sensitive queries
- High-priority searches
- Acceptable bandwidth overhead
```

**Example timeline:**
```
Time: 0s    → Launch 5 walkers in parallel
Time: 0.1s  → Each walker at hop 10
Time: 0.5s  → Walker 3 finds target at hop 50
            → Terminate all walkers immediately

Total time: 0.5 seconds
Total messages: 5 × 50 = 250 (early termination)
→ 20x faster than single walk!
```

---

**Scenario 3: Adaptive Multiple Walks (5 walkers with heuristics)**
```
Configuration:
- 5 walkers with weighted selection
- Diverse initial directions
- Check-back every 10 hops

Pros:
✅ Fastest: Intelligent routing
✅ Highest success rate: ~95%
✅ Efficient termination

Cons:
❌ Messages: 300-500 (variable)
❌ Complex implementation
❌ Requires node metadata

Use case:
- Production systems
- Critical rare data searches
- Acceptable complexity
```

**Example timeline:**
```
Time: 0s    → Launch 5 diverse walkers
Time: 0.05s → Walkers at hop 5
            → Check-back: No result yet
Time: 0.15s → Walker 2 finds at hop 15
            → Broadcast termination
            → Other walkers stop

Total time: 0.15 seconds
Total messages: ~75 (very efficient!)
→ 66x faster than single walk!
```

---

**C. Trade-off formula**

**Expected time to find:**
```
Single walker:
E[T_single] = E[hops] × RTT_per_hop
            = (1/p) × RTT
            where p = probability per node

Multiple walkers (k walkers):
E[T_multi] = E[T_single] / k  (approximate)
           = (1/p) × RTT / k

Speedup ≈ k (number of walkers)
```

**Expected messages:**
```
Single walker:
E[M_single] = E[hops] = 1/p

Multiple walkers (without early termination):
E[M_multi] = k × E[hops] = k/p

Multiple walkers (with early termination):
E[M_multi] ≈ k × E[hops_first_success]
           ≈ k × (1/kp)  (approximate)
           ≈ 1/p (same as single!)

→ Early termination is critical!
```

**Example calculation:**
```
Given:
- Replication ratio p = 0.001 (0.1%)
- RTT per hop = 10ms
- Number of walkers k = 5

Single walker:
E[T] = (1/0.001) × 10ms = 10,000ms = 10s
E[M] = 1/0.001 = 1,000 messages

Multiple walkers (with early termination):
E[T] = 10s / 5 = 2s
E[M] = 1,000 messages (distributed across 5 walkers)
      ≈ 200 messages (actual, due to early stop)

→ Time: 5x faster
→ Messages: Similar or better (with early termination)

Phần 5: Đề xuất giải pháp tối ưu
A. Hybrid Approach: k-Random Walks with Adaptive TTL
pythonclass OptimalRareDataSearch:
    def __init__(self):
        self.phase_config = [
            {'walkers': 3, 'ttl_per_walker': 50},   # Phase 1: Quick scan
            {'walkers': 5, 'ttl_per_walker': 100},  # Phase 2: Medium search
            {'walkers': 10, 'ttl_per_walker': 200}  # Phase 3: Deep search
        ]
    
    def search(self, filename, source_node):
        for phase in self.phase_config:
            result = self.execute_phase(
                filename, 
                source_node,
                num_walkers=phase['walkers'],
                ttl=phase['ttl_per_walker']
            )
            
            if result['found']:
                return result
            
            # Đợi một chút trước khi escalate
            sleep(0.5)
        
        return {'found': False}
    
    def execute_phase(self, filename, source, num_walkers, ttl):
        # Launch walkers with diverse directions
        walkers = self.launch_diverse_walkers(source, num_walkers)
        
        # Parallel execution with early termination
        results = parallel_execute_with_termination(walkers, ttl)
        
        return aggregate_results(results)
```

**Ưu điểm:**
```
✅ Escalating search: Bắt đầu nhẹ, tăng dần nếu cần
✅ Balance: Time vs Messages
✅ Adaptive: Dừng sớm nếu tìm thấy
```

**Example execution:**
```
Phase 1 (0-0.5s):
├─ Launch 3 walkers, TTL=50 each
├─ Total messages: ≤ 150
└─ Result: Not found

Phase 2 (0.5-1.5s):
├─ Launch 5 walkers, TTL=100 each
├─ Total messages: ≤ 500
└─ Result: FOUND at walker 2, hop 73!
└─ Total actual messages: 3×50 + 5×73 = 515

Total time: 1.5s
Success: YES
Messages: 515 (acceptable)

B. Learning-based approach
pythonclass LearningWalker:
    def __init__(self):
        self.node_success_history = {}  # Track which nodes often have rare files
    
    def weighted_selection(self, neighbors):
        """
        Chọn neighbor dựa trên lịch sử thành công
        """
        scores = []
        for neighbor in neighbors:
            # Base score
            score = 1.0
            
            # Bonus nếu neighbor từng có rare files
            if neighbor.id in self.node_success_history:
                score += self.node_success_history[neighbor.id] * 5
            
            scores.append(score)
        
        return weighted_random_choice(neighbors, scores)
    
    def update_history(self, path, found_node):
        """
        Sau khi tìm thấy, update history
        """
        for node in path:
            if node.id not in self.node_success_history:
                self.node_success_history[node.id] = 0
            
            # Nodes gần found_node có score cao hơn
            distance = path_distance(node, found_node)
            self.node_success_history[node.id] += 1.0 / (distance + 1)
```

**Lợi ích:**
```
✅ Học từ quá khứ
✅ Cải thiện qua thời gian
✅ Tận dụng clustering (rare files thường cluster)
```

---

#### **Phần 6: Tóm tắt và khuyến nghị**

**So sánh tổng quan:**

| Approach | Time | Messages | Success (rare) | Complexity | Best for |
|----------|------|----------|----------------|------------|----------|
| **Flooding** | ✅ Fast (3 hops) | ❌ High (1,110) | ❌ Low (8%) | ⚠️ Medium | Popular files |
| **Single Random Walk** | ❌ Slow (1,000 hops) | ✅ Low (1,000) | ⚠️ Medium (63%) | ✅ Simple | Low priority, constrained bandwidth |
| **Multiple Random Walks** | ⚠️ Medium (200 hops) | ⚠️ Medium (500) | ✅ High (92%) | ⚠️ Medium | Balanced approach |
| **Adaptive Multiple Walks** | ✅ Fast (50 hops) | ⚠️ Medium (300) | ✅ Very High (95%) | ❌ Complex | Production, critical searches |
| **Hybrid Approach** | ✅ Fast (avg 100) | ⚠️ Medium (400) | ✅ High (90%) | ⚠️ Medium | General purpose |

---

**Khuyến nghị theo use case:**

**1. Rare data, time-sensitive:**
```
✅ Use: Adaptive Multiple Walks (5-10 walkers)
Configuration:
- 5 walkers initially
- Diverse initial directions
- TTL = 100 per walker
- Check-back every 10 hops
- Weighted selection if possible

Expected:
- Time: 1-2 seconds
- Messages: 300-500
- Success: 90-95%
```

**2. Rare data, bandwidth-constrained:**
```
✅ Use: Single Random Walk with high TTL
Configuration:
- 1 walker
- TTL = 2,000
- Simple uniform random

Expected:
- Time: 10-20 seconds
- Messages: 1,000-2,000
- Success: 85%
```

**3. Rare data, balanced:**
```
✅ Use: Hybrid approach (escalating)
Configuration:
- Phase 1: 3 walkers × 50 TTL
- Phase 2: 5 walkers × 100 TTL
- Phase 3: 10 walkers × 200 TTL

Expected:
- Time: 2-5 seconds (average)
- Messages: 300-800 (average)
- Success: 90%
```

---

**Kết luận:**

Đối với rare data trong P2P không cấu trúc:

1. **Random Walk** là lựa chọn tốt hơn **Flooding** vì:
   - Giảm drastically network congestion
   - Có thể tìm kiếm sâu với chi phí thấp
   - Phù hợp với rare data (low replication)

2. **Multiple Random Walks** cải thiện đáng kể:
   - Time: Giảm k lần (k = số walkers)
   - Success rate: Tăng exponentially
   - Messages: Chỉ tăng tuyến tính (với early termination)

3. **Trade-off chính:**
```
   More walkers:
   ✅ Faster search
   ✅ Higher success rate
   ❌ More messages
   ❌ More coordination complexity

Sweet spot: 5-10 parallel walkers với adaptive TTL

Balance tốt nhất giữa time và messages
Success rate cao (>90%)
Practical implementation