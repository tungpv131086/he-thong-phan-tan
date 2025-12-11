# Câu 3: Chord DHT - Distributed Hash Table

> **Chương:** 2 - Kiến trúc Hệ thống Phân tán  
> **Độ khó:** ⭐⭐⭐⭐ (Khó)  
> **Thời gian đọc:** ~25 phút

---

## 📋 Mục lục

- [Đề bài](#đề-bài)
- [Phần 1: Cơ bản về Chord](#phần-1-cơ-bản-về-chord)
- [Phần 2: Tính toán Successor](#phần-2-tính-toán-successor)
- [Phần 3: Routing với Finger Table](#phần-3-routing-với-finger-table)
- [Phần 4: Phân tích Hiệu năng](#phần-4-phân-tích-hiệu-năng)
- [Tóm tắt](#tóm-tắt)

---

## 📋 Đề bài

Cho một hệ thống Chord DHT (Distributed Hash Table) với **m = 5** (không gian định danh 0–31).

Các node hiện có trong ring: **{1, 4, 9, 11, 14, 18, 20, 21, 28}**

**Yêu cầu:**

1. **Xác định successor** cho các keys:
   - key = 7
   - key = 22
   - key = 30

2. **Mô tả quá trình routing** khi node 9 tìm kiếm key = 3:
   - Sử dụng finger table
   - Liệt kê các bước nhảy (hops)
   - Giải thích tại sao hiệu quả hơn routing tuần tự

---

## 💡 Bài giải

### Phần 1: Cơ bản về Chord

#### A. Không gian định danh
```
Chord Ring với m = 5:
────────────────────────────────────────
Identifier space: 0 to 2^m - 1
                = 0 to 2^5 - 1
                = 0 to 31

Total identifiers: 32 (0, 1, 2, ..., 31)
```

#### B. Chord Ring Visualization
```
                        0/32
                         •
                    31 ╱   ╲ 1 ●
                   ╱         ╲
              30 ●           ● 2
                 │           │
             29  │           │  3
                 │           │
             28 ●│           │● 4
                 │           │
             27  │           │  5
                 │           │
             26  │           │  6
                 │           │
             25  │           │  7
                 │           │
             24  │           │  8
                 │           │
             23  │           │● 9
                 │           │
             22  │           │  10
                ●│           │● 11
            21   │           │
                 │           │  12
            20 ●─┘           └─  13
                 ╲         ╱
              19  ╲       ╱ 14 ●
                18 ●─────● 15
                    17 16

Nodes present (●):
{1, 4, 9, 11, 14, 18, 20, 21, 28}

Total: 9 nodes out of 32 possible positions
```

#### C. Successor Definition

**Định nghĩa:**
```
successor(k) = node n where:
- n is the first node ≥ k in the ring
- If no node ≥ k exists, wrap around to first node

Formula:
successor(k) = min{n ∈ Nodes | n ≥ k}
             OR first node if no such n exists
```

**Tại sao quan trọng:**
- Mỗi key được lưu tại node successor của nó
- Đảm bảo mọi key đều có một "chủ nhân"
- Khi node join/leave, chỉ cần di chuyển keys giữa successor và predecessor

---

### Phần 2: Tính toán Successor

#### A. Key = 7
```
Question: successor(7) = ?

Step 1: Tìm node đầu tiên ≥ 7
────────────────────────────────────────
Nodes: {1, 4, 9, 11, 14, 18, 20, 21, 28}
              ↑
              9 is first node ≥ 7

Answer: successor(7) = 9 ✅
```

**Visualization:**
```
        7 (key position)
         ↓
    ... 6  7  8  9 ...
            ↗ ↑
      (no node) │
                │
           Node 9 (successor)
```

**Giải thích:**
- Key 7 nằm giữa node 4 và node 9
- Theo quy tắc: successor = first node ≥ key
- Node 9 là node đầu tiên ≥ 7
- Vậy key 7 được lưu tại **node 9**

---

#### B. Key = 22
```
Question: successor(22) = ?

Step 1: Tìm node đầu tiên ≥ 22
────────────────────────────────────────
Nodes: {1, 4, 9, 11, 14, 18, 20, 21, 28}
                                    ↑
                              28 is first node ≥ 22

Answer: successor(22) = 28 ✅
```

**Visualization:**
```
    20  21  22  23  24  25  26  27  28
     ●   ●   ↑                       ●
             │                       ↑
        key 22              successor(22) = 28
```

**Giải thích:**
- Key 22 nằm giữa node 21 và node 28
- Node đầu tiên ≥ 22 là node 28
- Vậy key 22 được lưu tại **node 28**

---

#### C. Key = 30
```
Question: successor(30) = ?

Step 1: Tìm node đầu tiên ≥ 30
────────────────────────────────────────
Nodes: {1, 4, 9, 11, 14, 18, 20, 21, 28}
                                    ↑
                    28 < 30 (không thỏa)

Step 2: Không có node ≥ 30
→ Wrap around to first node

Answer: successor(30) = 1 ✅
```

**Visualization (Ring wrap-around):**
```
             0/32
              ●
         31 ╱   ╲ 1 ●  ← successor(30)
        ╱         ╲
    30 ●           ● 2
     ↑ (key)
    
No node between 30 and 32
→ Wrap to first node = 1
```

**Giải thích:**
- Key 30 > tất cả nodes hiện có (max = 28)
- Theo quy tắc wrap-around của ring
- Successor = node đầu tiên trong ring
- Vậy key 30 được lưu tại **node 1**

---

#### D. Summary Table
```
╔═══════════════════════════════════════════╗
║         SUCCESSOR CALCULATIONS            ║
╠═══════════════════════════════════════════╣
║                                           ║
║  Key │ Successor │ Explanation           ║
║ ═════╪═══════════╪═════════════════════  ║
║      │           │                       ║
║   7  │     9     │ First node ≥ 7       ║
║      │           │ 7 ∈ (4, 9]           ║
║      │           │                       ║
║  22  │    28     │ First node ≥ 22      ║
║      │           │ 22 ∈ (21, 28]        ║
║      │           │                       ║
║  30  │     1     │ No node ≥ 30         ║
║      │           │ Wrap to first node   ║
║      │           │ 30 ∈ (28, 1]         ║
║      │           │                       ║
╚═══════════════════════════════════════════╝
```

---

### Phần 3: Routing với Finger Table

#### A. Finger Table của Node 9

**Công thức finger table:**
```
For node n with m-bit identifier:
finger[i] = successor(n + 2^i)  where i ∈ [0, m-1]

For node 9 (m = 5):
finger[0] = successor(9 + 2^0) = successor(10)
finger[1] = successor(9 + 2^1) = successor(11)
finger[2] = successor(9 + 2^2) = successor(13)
finger[3] = successor(9 + 2^3) = successor(17)
finger[4] = successor(9 + 2^4) = successor(25)
```

**Tính toán từng entry:**
```
Node 9 Finger Table:
═══════════════════════════════════════════════

finger[0] = successor(9 + 1) = successor(10)
Nodes: {1, 4, 9, 11, 14, 18, 20, 21, 28}
                    ↑
First node ≥ 10 = 11
finger[0] = 11

────────────────────────────────────────────────

finger[1] = successor(9 + 2) = successor(11)
First node ≥ 11 = 11
finger[1] = 11

────────────────────────────────────────────────

finger[2] = successor(9 + 4) = successor(13)
First node ≥ 13 = 14
finger[2] = 14

────────────────────────────────────────────────

finger[3] = successor(9 + 8) = successor(17)
First node ≥ 17 = 18
finger[3] = 18

────────────────────────────────────────────────

finger[4] = successor(9 + 16) = successor(25)
First node ≥ 25 = 28
finger[4] = 28
```

**Finger Table hoàn chỉnh:**
```
╔═══════════════════════════════════════════════╗
║       NODE 9 FINGER TABLE                     ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  i │ start │ interval    │ successor │ node  ║
║ ═══╪═══════╪═════════════╪═══════════╪══════ ║
║    │       │             │           │       ║
║  0 │  10   │ [10, 11)    │    11     │  11   ║
║  1 │  11   │ [11, 13)    │    11     │  11   ║
║  2 │  13   │ [13, 17)    │    14     │  14   ║
║  3 │  17   │ [17, 25)    │    18     │  18   ║
║  4 │  25   │ [25, 9)     │    28     │  28   ║
║    │       │             │           │       ║
╚═══════════════════════════════════════════════╝

Coverage visualization:
0──────────────────────────────31
   │ │    │        │            │
   11 14   18       28          (wraps to 9)
   ↑  ↑    ↑        ↑
  2^0 2^2  2^3      2^4 distance from node 9
```

---

#### B. Routing: Node 9 tìm Key = 3

**Algorithm:**
```python
def find_successor(node, key):
    """
    Find successor of key starting from node
    """
    if key in (node, node.successor]:
        return node.successor
    else:
        # Find closest preceding node in finger table
        n0 = closest_preceding_node(node, key)
        return n0.find_successor(key)

def closest_preceding_node(node, key):
    """
    Return closest finger that precedes key
    """
    for i in range(m-1, -1, -1):  # Check from largest to smallest
        if node.finger[i] in (node, key):
            return node.finger[i]
    return node
```

**Step-by-step routing:**
```
═══════════════════════════════════════════════
STEP 0: Node 9 wants to find key = 3
═══════════════════════════════════════════════

Current node: 9
Target key: 3
Question: Is key ∈ (9, successor(9)]?

Calculate successor(9):
- From finger[0] = 11
- So key ∈ (9, 11]?
- 3 ∈ (9, 11]? NO (3 < 9)

Action: Forward to closest preceding finger
```
```
═══════════════════════════════════════════════
STEP 1: Check finger table of node 9
═══════════════════════════════════════════════

Node 9 finger table: {11, 11, 14, 18, 28}

Question: Which finger is closest to key=3 but doesn't overshoot?

Check fingers from largest to smallest:
- finger[4] = 28: Is 28 in (9, 3)? YES ✅
  (Circular: 9 → 28 → 0 → 3)

Decision: Forward to node 28
```
```
═══════════════════════════════════════════════
STEP 2: Node 28 receives query for key = 3
═══════════════════════════════════════════════

Current node: 28
Target key: 3
Question: Is key ∈ (28, successor(28)]?

Calculate successor(28):
Nodes after 28: {28, ..., 0, 1, 4, ...}
- Wrap around: successor(28) = 1
- Is 3 ∈ (28, 1]? 
- Circular: 28 → 29 → 30 → 31 → 0 → 1
- 3 > 1, so NO

Check finger table of node 28:
finger[0] = successor(29) = 1
- Is 1 in (28, 3)? YES ✅

Decision: Forward to node 1
```
```
═══════════════════════════════════════════════
STEP 3: Node 1 receives query for key = 3
═══════════════════════════════════════════════

Current node: 1
Target key: 3
Question: Is key ∈ (1, successor(1)]?

Calculate successor(1):
- Next node after 1 = 4
- Is 3 ∈ (1, 4]? YES ✅

Action: Forward to successor = node 4
```
```
═══════════════════════════════════════════════
STEP 4: Node 4 returns result
═══════════════════════════════════════════════

Node 4 is responsible for key = 3
(because 3 ∈ (1, 4])

Return: "Node 4 has key 3"
```

**Complete Routing Path:**
```
┌─────────────────────────────────────────────┐
│  ROUTING SUMMARY                            │
├─────────────────────────────────────────────┤
│                                             │
│  Hop 1: Node 9  → Query key 3              │
│         ├─ Check finger table               │
│         └─ Forward to finger[4] = 28        │
│         (Jump: +19 positions)               │
│                                             │
│  Hop 2: Node 28 → Receive query            │
│         ├─ Check finger table               │
│         └─ Forward to finger[0] = 1         │
│         (Jump: wrap around, +5 positions)   │
│                                             │
│  Hop 3: Node 1  → Receive query            │
│         ├─ Check: 3 ∈ (1, 4]? YES          │
│         └─ Forward to successor = 4         │
│         (Jump: +3 positions)                │
│                                             │
│  Hop 4: Node 4  → Return data              │
│         └─ Key 3 found! ✅                  │
│                                             │
├─────────────────────────────────────────────┤
│  Total hops: 3 (9 → 28 → 1 → 4)           │
│  Total nodes contacted: 4                   │
└─────────────────────────────────────────────┘
```

**Visual Representation:**
```
             Ring (0-31)
                 
    0 ●─────────────────────● 16
      │                     │
      │   ┌─────────────┐  │
      │   │   TARGET    │  │
    1 ●   │   key = 3   │  │
      │   └──────↑──────┘  │
      │          │          │
    4 ●──────────┘          │
      │ ↑                   │
      │ └── HOP 3           │
    8 │     (from 1)        │
      │                     │
    9 ●◄────┐               │
      │ START│              │
      │      │              │
   11 │      │              │
      │      │              │
   14 │      │              │
      │      │              │
   18 │      │              │
      │      │              │
   20 │      │              │
   21 │      │              │
      │      │              │
   28 ●◄─────┘              │
      │  HOP 1               │
      │  (jump +19)         │
   31 │                     │
      │                     │
    0 └─────────────────────┘

Path: 9 → 28 → 1 → 4
Distance: 3 hops ✅
```

---

#### C. Tại sao hiệu quả hơn routing tuần tự?

**Comparison:**
```
╔═══════════════════════════════════════════════╗
║     SEQUENTIAL vs FINGER TABLE ROUTING        ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Method          │ Hops │ Nodes   │ Time     ║
║                  │      │ Visited │          ║
║ ═════════════════╪══════╪═════════╪═════════ ║
║                                               ║
║  Sequential      │  9   │   10    │ O(N)     ║
║  (ask next node) │      │         │          ║
║                  │      │         │          ║
║  Finger Table    │  3   │    4    │ O(log N) ║
║  (smart jumps)   │      │         │          ║
║                  │      │         │          ║
║  Improvement     │ 3x   │  2.5x   │ ✅       ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

**Sequential Routing (Baseline):**
```
Node 9 → Node 11 → Node 14 → Node 18 → 
Node 20 → Node 21 → Node 28 → Node 1 → Node 4

Total: 9 hops ❌
```

**Finger Table Routing (Optimized):**
```
Node 9 → Node 28 → Node 1 → Node 4

Total: 3 hops ✅ (3x faster!)
```

**Why Finger Table is Better:**

1. **Exponential Coverage**
```
   From node n, finger table covers:
   - finger[0]: n + 1     (next node)
   - finger[1]: n + 2     (skip 1)
   - finger[2]: n + 4     (skip 3)
   - finger[3]: n + 8     (skip 7)
   - finger[4]: n + 16    (skip 15)
   
   Each finger doubles the distance!
```

2. **Binary Search-like**
```
   Similar to binary search in sorted array:
   - Don't check every element
   - Jump to middle, then half of half
   - O(log N) complexity ✅
```

3. **Scalability**
```
   Number of nodes: 32    1024    1M      1B
   ─────────────────────────────────────────────
   Sequential:       32    1024    1M      1B ❌
   Finger table:      5      10    20      30 ✅
   
   Improvement:      6x    100x   50Kx    33Mx 🚀
```

---

### Phần 4: Phân tích Hiệu năng

#### A. Complexity Analysis
```
╔═══════════════════════════════════════════════╗
║          CHORD PERFORMANCE ANALYSIS           ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Operation       │ Complexity │ Notes         ║
║ ═════════════════╪════════════╪═════════════  ║
║                                               ║
║  Lookup          │ O(log N)   │ Using fingers ║
║  Insert key      │ O(log N)   │ Find + store  ║
║  Delete key      │ O(log N)   │ Find + remove ║
║                                               ║
║  Node join       │ O(log² N)  │ Update tables ║
║  Node leave      │ O(log² N)  │ Transfer keys ║
║                                               ║
║  Storage per node│ O(log N)   │ Finger table  ║
║  Messages/lookup │ O(log N)   │ Hop count     ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

**Proof of O(log N) lookup:**
```
Theorem: Any lookup requires at most log₂(N) hops

Proof:
1. Identifier space: [0, 2^m - 1]
2. Finger table has m entries
3. Each finger[i] covers distance 2^i

At each hop:
- Distance to target is halved (at least)
- Similar to binary search

Example with N = 32 (m = 5):
- Worst case: 5 hops
- log₂(32) = 5 ✅

Example with N = 1024 (m = 10):
- Worst case: 10 hops
- log₂(1024) = 10 ✅
```

#### B. Load Balancing
```
Keys distribution (with N=9 nodes, M=32 keys):

Node  │ Range        │ Keys    │ Load
──────┼──────────────┼─────────┼──────
  1   │ (28, 1]      │ 4 keys  │ 12.5%
  4   │ (1, 4]       │ 3 keys  │  9.4%
  9   │ (4, 9]       │ 5 keys  │ 15.6%
 11   │ (9, 11]      │ 2 keys  │  6.3%
 14   │ (11, 14]     │ 3 keys  │  9.4%
 18   │ (14, 18]     │ 4 keys  │ 12.5%
 20   │ (18, 20]     │ 2 keys  │  6.3%
 21   │ (20, 21]     │ 1 key   │  3.1%
 28   │ (21, 28]     │ 7 keys  │ 21.9%

Average: 3.56 keys/node
Max: 7 keys (node 28)
Min: 1 key (node 21)
Variance: Moderate

With consistent hashing:
- Load is roughly balanced ✅
- Adding node redistributes ~1/N keys
- Removing node affects only successor
```

#### C. Fault Tolerance
```
Scenario: Node 28 fails

Impact:
1. Keys stored at node 28:
   - Range (21, 28]
   - 7 keys affected
   - Transferred to successor(28) = node 1

2. Finger tables pointing to 28:
   - Node 9: finger[4] = 28 → Update to 1
   - Node 11: finger[4] = 28 → Update to 1
   - Node 14: finger[3] = 28 → Update to 1
   - Node 18: finger[3] = 28 → Update to 1
   - Node 20: finger[2] = 28 → Update to 1

3. Successor pointers:
   - Node 21: successor = 28 → Update to 1

Recovery:
✅ Keys not lost (stored at successor)
✅ Finger tables updated lazily
✅ System continues operating
⚠️ Temporary performance degradation

Chord's stabilization protocol:
- Nodes periodically check successors
- Fix finger tables incrementally
- Recover to optimal state in O(N log N) time
```

---

## 📊 Tóm tắt

### Key Points

- ✅ **Chord DHT**: Distributed hash table with O(log N) lookup
- ✅ **Successor function**: First node ≥ key (with wrap-around)
- ✅ **Finger table**: m entries covering exponential distances
- ✅ **Routing**: Binary-search-like, 3 hops vs 9 sequential
- ✅ **Scalability**: Handles 1 billion nodes with 30 hops max

### Kết quả bài toán
```
╔═══════════════════════════════════════════╗
║            ANSWERS SUMMARY                ║
╠═══════════════════════════════════════════╣
║                                           ║
║  1. Successor calculations:               ║
║     ├─ successor(7)  = 9                 ║
║     ├─ successor(22) = 28                ║
║     └─ successor(30) = 1 (wrap-around)   ║
║                                           ║
║  2. Routing (node 9 → key 3):            ║
║     ├─ Hop 1: 9  → 28 (finger[4])       ║
║     ├─ Hop 2: 28 → 1  (finger[0])       ║
║     ├─ Hop 3: 1  → 4  (successor)       ║
║     └─ Total: 3 hops (vs 9 sequential)   ║
║                                           ║
║  3. Efficiency:                           ║
║     ├─ Finger table: O(log N)            ║
║     ├─ Sequential: O(N)                  ║
║     └─ Improvement: 3x for N=9           ║
║                     50,000x for N=1M 🚀  ║
║                                           ║
╚═══════════════════════════════════════════╝
```

### Trade-offs

**Advantages:**
- ✅ Logarithmic lookup time
- ✅ Scalable to millions of nodes
- ✅ Fault tolerant (successor backup)
- ✅ Load balanced (consistent hashing)
- ✅ Decentralized (no single point of failure)

**Disadvantages:**
- ⚠️ Churn handling (nodes join/leave frequently)
- ⚠️ Finger table maintenance overhead
- ⚠️ Not optimal for range queries
- ⚠️ Network latency not considered (logical routing)

### Applications

- **BitTorrent DHT**: Peer discovery
- **Amazon Dynamo**: Key-value store
- **IPFS**: Content addressing
- **Bitcoin**: Peer network (similar concept)
- **Cassandra**: Distributed database (ring topology)

---

## 🔗 Tài liệu tham khảo

### Papers
- **"Chord: A Scalable Peer-to-peer Lookup Service"** - Stoica et al., SIGCOMM 2001
- **"Consistent Hashing and Random Trees"** - Karger et al., 1997

### Books
- **Distributed Systems** - Tanenbaum & Van Steen (Chapter 5)
- **Designing Data-Intensive Applications** - Kleppmann (Chapter 6)

### Online Resources
- [Chord Visualization](https://www.pdl.cmu.edu/Chord/)
- [MIT 6.824 Lecture Notes](https://pdos.csail.mit.edu/6.824/)

---

## 🧭 Navigation

**[⬅️ Câu 2: Kiến trúc 3 tầng](./cau-2-kien-truc-3-tang.md)** | **[📚 Quay lại Chương 2](./README.md)** | **[➡️ Câu 4: P2P Flooding](./cau-4-p2p-flooding.md)**

---

*Cập nhật lần cuối: 11/12/2025*