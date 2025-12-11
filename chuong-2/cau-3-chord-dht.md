CHƯƠNG 2 - CÂU 3
Đề bài:
Trong vòng Chord với m = 5 và các nút hiện có {1, 4, 9, 11, 14, 18, 20, 21, 28}, hãy:

Xác định succ(7), succ(22) và succ(30).
Giả sử node 9 thực hiện tra cứu key = 3, hãy mô tả chi tiết các bước chuyển tiếp yêu cầu qua các shortcut (theo hình 2.19) cho đến khi tìm được node chịu trách nhiệm.


BÀI GIẢI:
Phần 1: Kiến thức nền tảng về Chord DHT
A. Chord Ring (Vòng Chord)
Chord là một giao thức Distributed Hash Table (DHT) sử dụng consistent hashing để phân bổ dữ liệu trên các node trong mạng P2P.
Thông số:

m = 5: Số bit để định danh node và key
Không gian địa chỉ: 0 đến 2^m - 1 = 2^5 - 1 = 31
Tổng số vị trí có thể: 32 (từ 0 đến 31)

Các node hiện có: {1, 4, 9, 11, 14, 18, 20, 21, 28}
Biểu diễn vòng Chord:
                    0/32
                     │
        28 ────────  │  ────────  1
                    │
       21 ─────────     ───────── 4
                   │   │
      20 ──────────     ────────── 9
                  │     │
     18 ─────────   Chord  ──────── 11
                  │  Ring  │
                 │         │
                14 ─────────
B. Hàm successor (succ)
Định nghĩa: succ(k) là node đầu tiên có ID ≥ k khi đi theo chiều kim đồng hồ trên vòng Chord.
Công thức:
succ(k) = min{n ∈ nodes | n ≥ k} 
          hoặc min{nodes} nếu k > max{nodes}
Trách nhiệm của node:

Node n chịu trách nhiệm lưu trữ tất cả các key k thỏa mãn: pred(n) < k ≤ n
pred(n): node đứng ngay trước n trên vòng


Phần 2: Xác định succ(7), succ(22), succ(30)
Danh sách nodes sắp xếp: 1, 4, 9, 11, 14, 18, 20, 21, 28

CÂU 2.1: Xác định succ(7)
Phân tích:

Tìm node nhỏ nhất có ID ≥ 7
Các node có ID ≥ 7: {9, 11, 14, 18, 20, 21, 28}
Node nhỏ nhất trong số này: 9

Kết quả:
succ(7) = 9
Giải thích chi tiết:
Vòng Chord:
... ─ 4 ─ [vị trí 5,6,7,8] ─ 9 ─ 11 ─ ...
              ↑
         Key 7 rơi vào đây
         
Node 9 là node đầu tiên >= 7 khi đi theo chiều kim đồng hồ
→ Key 7 được lưu trữ tại node 9
Các key mà node 9 chịu trách nhiệm:
pred(9) = 4
Node 9 lưu trữ: (4, 9] = {5, 6, 7, 8, 9}

CÂU 2.2: Xác định succ(22)
Phân tích:

Tìm node nhỏ nhất có ID ≥ 22
Các node có ID ≥ 22: {28}
Node nhỏ nhất: 28

Kết quả:
succ(22) = 28
Giải thích chi tiết:
Vòng Chord:
... ─ 21 ─ [vị trí 22,23,24,25,26,27] ─ 28 ─ 1 ─ ...
               ↑
          Key 22 rơi vào đây
          
Node 28 là node đầu tiên >= 22
→ Key 22 được lưu trữ tại node 28
Các key mà node 28 chịu trách nhiệm:
pred(28) = 21
Node 28 lưu trữ: (21, 28] = {22, 23, 24, 25, 26, 27, 28}

CÂU 2.3: Xác định succ(30)
Phân tích:

Tìm node nhỏ nhất có ID ≥ 30
Không có node nào có ID ≥ 30 (max node = 28)
Vòng Chord là vòng tròn → quay lại đầu vòng
Node đầu tiên trong vòng: 1

Kết quả:
succ(30) = 1
Giải thích chi tiết:
Vòng Chord (hiển thị wrap-around):
28 ─ [29, 30, 31, 0] ─ 1 ─ 4 ─ ...
         ↑
    Key 30 rơi vào đây
    
Không có node >= 30 trong [0, 31]
→ Quay vòng lại: succ(30) = node đầu tiên = 1
Các key mà node 1 chịu trách nhiệm:
pred(1) = 28
Node 1 lưu trữ: (28, 1] = {29, 30, 31, 0, 1}
(31 + 1 = 0 mod 32, vòng quay lại)

Tóm tắt Phần 1:
KeySuccessorNode chịu trách nhiệmGiải thích79Node 9Node đầu tiên >= 72228Node 28Node đầu tiên >= 22301Node 1Wrap-around, quay lại đầu vòng

Phần 3: Finger Table và Routing trong Chord
A. Khái niệm Finger Table
Mỗi node n duy trì một finger table với m entries (m = 5 → 5 entries).
Entry thứ i (i = 0, 1, 2, ..., m-1):
finger[i].start = (n + 2^i) mod 2^m
finger[i].node = succ(finger[i].start)
Mục đích:

Tăng tốc độ routing từ O(N) xuống O(log N)
Mỗi finger "nhảy" một khoảng cách tăng theo lũy thừa 2


B. Xây dựng Finger Table cho Node 9
Node 9 với m = 5:
ifinger[i].start = (9 + 2^i) mod 32finger[i].intervalfinger[i].node = succ(start)0(9 + 2^0) mod 32 = 10[10, 11)succ(10) = 111(9 + 2^1) mod 32 = 11[11, 13)succ(11) = 112(9 + 2^2) mod 32 = 13[13, 17)succ(13) = 143(9 + 2^3) mod 32 = 17[17, 25)succ(17) = 184(9 + 2^4) mod 32 = 25[25, 9)succ(25) = 28
Finger Table của Node 9:
Node 9 Finger Table:
┌───┬───────┬──────────────┬──────┐
│ i │ start │  interval    │ node │
├───┼───────┼──────────────┼──────┤
│ 0 │  10   │  [10, 11)    │  11  │
│ 1 │  11   │  [11, 13)    │  11  │
│ 2 │  13   │  [13, 17)    │  14  │
│ 3 │  17   │  [17, 25)    │  18  │
│ 4 │  25   │  [25, 9)     │  28  │
└───┴───────┴──────────────┴──────┘
Giải thích:

Finger 0: Nhảy +1 → node 11 (gần nhất)
Finger 1: Nhảy +2 → vẫn node 11
Finger 2: Nhảy +4 → node 14
Finger 3: Nhảy +8 → node 18
Finger 4: Nhảy +16 → node 28 (xa nhất)


Phần 4: Tra cứu key = 3 từ node 9
CÂU 2.2: Mô tả chi tiết các bước routing
Mục tiêu: Node 9 cần tìm node chịu trách nhiệm cho key = 3
Bước chuẩn bị:

Xác định node chịu trách nhiệm key = 3:

   succ(3) = 4
   → Node 4 chịu trách nhiệm lưu trữ key 3

Thuật toán routing:

   Từ node n, để tìm key k:
   - Nếu k ∈ (n, successor(n)]: Trả về successor(n)
   - Ngược lại: Tìm node n' trong finger table sao cho:
     n' là node xa nhất mà vẫn < k
     → Forward request đến n'

BƯỚC 1: Node 9 xử lý request
Kiểm tra:
Current node: 9
Target key: 3
Successor của node 9: succ(9) = 11

Kiểm tra: key 3 có thuộc (9, 11] không?
→ KHÔNG (vì 3 < 9, cần wrap-around)
Tìm node tiếp theo trong finger table:
Finger Table của Node 9:
- finger[0].node = 11 (> 3? Có → Không hợp lệ)
- finger[1].node = 11 (> 3? Có → Không hợp lệ)
- finger[2].node = 14 (> 3? Có → Không hợp lệ)
- finger[3].node = 18 (> 3? Có → Không hợp lệ)
- finger[4].node = 28 (> 3? Có → ???)
Lưu ý về wrap-around:
Trên vòng Chord, từ node 9 đến key 3:
9 → 11 → ... → 28 → 1 → 4 (key 3 nằm ở đây)

Do key 3 < 9, ta cần đi "vòng qua" 0
Node 28 là node lớn nhất < 32, gần với đích nhất
Quyết định:
Forward request đến finger[4].node = 28
(Node 28 là node xa nhất theo chiều dương)
Action:
Node 9 gửi: lookup(key=3) → Node 28

BƯỚC 2: Node 28 xử lý request
Xây dựng Finger Table cho Node 28:
ifinger[i].start = (28 + 2^i) mod 32finger[i].node = succ(start)0(28 + 1) mod 32 = 29succ(29) = 11(28 + 2) mod 32 = 30succ(30) = 12(28 + 4) mod 32 = 0succ(0) = 13(28 + 8) mod 32 = 4succ(4) = 44(28 + 16) mod 32 = 12succ(12) = 14
Node 28 Finger Table:
┌───┬───────┬──────────────┬──────┐
│ i │ start │  interval    │ node │
├───┼───────┼──────────────┼──────┤
│ 0 │  29   │  [29, 30)    │  1   │
│ 1 │  30   │  [30, 0)     │  1   │
│ 2 │   0   │  [0, 4)      │  1   │
│ 3 │   4   │  [4, 12)     │  4   │
│ 4 │  12   │  [12, 28)    │  14  │
└───┴───────┴──────────────┴──────┘
Kiểm tra:
Current node: 28
Target key: 3
Successor của node 28: succ(28) = 1

Kiểm tra: key 3 có thuộc (28, 1] không?
→ (28, 1] = {29, 30, 31, 0, 1} (wrap-around)
→ KHÔNG (3 không nằm trong khoảng này)
Tìm node tiếp theo:
Tìm node n' trong finger table sao cho n' là node lớn nhất mà < 3
(hoặc node gần key 3 nhất mà không vượt qua nó)

Xét các fingers:
- finger[0].node = 1 (< 3? Có ✓)
- finger[1].node = 1 (< 3? Có ✓)
- finger[2].node = 1 (< 3? Có ✓)
- finger[3].node = 4 (< 3? Không ✗, 4 > 3)
- finger[4].node = 14 (< 3? Không ✗)

Node 1 là lựa chọn tốt nhất (gần nhất mà < 3)
Action:
Node 28 gửi: lookup(key=3) → Node 1

BƯỚC 3: Node 1 xử lý request
Xây dựng Finger Table cho Node 1:
ifinger[i].start = (1 + 2^i) mod 32finger[i].node = succ(start)0(1 + 1) mod 32 = 2succ(2) = 41(1 + 2) mod 32 = 3succ(3) = 42(1 + 4) mod 32 = 5succ(5) = 93(1 + 8) mod 32 = 9succ(9) = 94(1 + 16) mod 32 = 17succ(17) = 18
Node 1 Finger Table:
┌───┬───────┬──────────────┬──────┐
│ i │ start │  interval    │ node │
├───┼───────┼──────────────┼──────┤
│ 0 │   2   │  [2, 3)      │  4   │
│ 1 │   3   │  [3, 5)      │  4   │
│ 2 │   5   │  [5, 9)      │  9   │
│ 3 │   9   │  [9, 17)     │  9   │
│ 4 │  17   │  [17, 1)     │  18  │
└───┴───────┴──────────────┴──────┘
Kiểm tra:
Current node: 1
Target key: 3
Successor của node 1: succ(1) = 4

Kiểm tra: key 3 có thuộc (1, 4] không?
→ (1, 4] = {2, 3, 4}
→ CÓ! Key 3 nằm trong khoảng này
Kết luận:
Node 1 xác định: succ(3) = 4
→ Node 4 chịu trách nhiệm key 3
Action:
Node 1 gửi: lookup(key=3) → Node 4 (final destination)

BƯỚC 4: Node 4 trả về kết quả
Node 4 nhận request lookup(key=3)
Node 4 kiểm tra: pred(4) = 1
Key 3 thuộc (1, 4] → Node 4 chịu trách nhiệm

Node 4 thực hiện:
1. Tra cứu key 3 trong local storage
2. Trả về dữ liệu (hoặc NOT_FOUND nếu không có)
Response path:
Node 4 → Node 1 → Node 28 → Node 9 (original requester)

Phần 5: Tóm tắt và phân tích
Tổng kết routing path:
┌────────────────────────────────────────────────────────────┐
│  ROUTING PATH: Node 9 tìm key = 3                         │
└────────────────────────────────────────────────────────────┘

Bước 1: Node 9 (start)
        ├─ Check: key 3 ∈ (9, 11]? → NO
        ├─ Lookup finger table
        └─ Forward to: Node 28 (finger[4], nhảy +16)
        
Bước 2: Node 28
        ├─ Check: key 3 ∈ (28, 1]? → NO
        ├─ Lookup finger table
        └─ Forward to: Node 1 (finger[0], nhảy +1)
        
Bước 3: Node 1
        ├─ Check: key 3 ∈ (1, 4]? → YES!
        └─ Forward to: Node 4 (successor)
        
Bước 4: Node 4 (destination)
        └─ Return data for key 3

TOTAL HOPS: 3 hops (9 → 28 → 1 → 4)

Sơ đồ trực quan:
        Chord Ring (m=5, N=32)
        
    0/32
     │
 28 ─┼─ 1  ←─── Bước 3: Check (1,4], tìm thấy!
     │   │              Forward to 4
     │   4  ←─── Bước 4: Đích cuối cùng (key=3)
     │   
 21─ │
     │   9  ←─── Bước 1: Start here
 20─ │            Forward to 28 (big jump)
     │   11
 18─ │   
     │   14

Path: 9 ──(+19)──> 28 ──(+5)──> 1 ──(+3)──> 4

Phân tích hiệu năng:
1. Độ phức tạp routing:
- Không có finger table: O(N) = 9 hops
  (phải đi qua tất cả 9 nodes: 9→11→14→18→20→21→28→1→4)
  
- Với finger table: O(log N) = 3 hops
  (nhảy theo lũy thừa 2: 9→28→1→4)
2. Khoảng cách tối ưu:
Từ node 9 đến node 4:
- Khoảng cách trực tiếp: |4 - 9| = 5 (hoặc 27 theo chiều ngược)
- Thực tế trên vòng: 9 → 11 → ... → 28 → 1 → 4

Finger table cho phép:
- Bước 1: Nhảy +19 (9→28) - Big jump
- Bước 2: Nhảy +5 (28→1) - Medium jump
- Bước 3: Nhảy +3 (1→4) - Small jump

→ Hiệu quả gấp 3 lần!
3. Lợi ích của finger table:
✅ Giảm số hops từ O(N) xuống O(log N)
✅ Mỗi hop "hẹp dần" khoảng cách với target
✅ Self-stabilizing: tự động điều chỉnh khi node join/leave

So sánh với các phương pháp khác:
Phương phápHopsComplexityBộ nhớLinear search9O(N)O(1) - chỉ cần biết successorChord với finger table3O(log N)O(m log N) = O(log² N)Flooding (P2P không cấu trúc)All nodesO(N)O(1) nhưng broadcast storm

Kết luận:
Chord DHT với finger table cung cấp:

Routing hiệu quả: O(log N) hops
Decentralized: Không cần server trung tâm
Scalable: Thêm/bớt node không ảnh hưởng performance nhiều
Load balanced: Keys phân bố đều nhờ consistent hashing

Trong ví dụ này:

succ(7) = 9: Node 9 chịu trách nhiệm key 7
succ(22) = 28: Node 28 chịu trách nhiệm key 22
succ(30) = 1: Node 1 chịu trách nhiệm key 30 (wrap-around)
Routing từ node 9 đến key 3: 3 hops (9 → 28 → 1 → 4)