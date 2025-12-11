CHƯƠNG 2 - CÂU 2
Đề bài:
Một hệ thống bán hàng trực tuyến được thiết kế theo kiến trúc 3 tầng (three-tiered architecture) gồm:

Client (UI layer): giao diện web cho khách hàng đặt hàng.
Application server (Processing layer): xử lý đơn hàng, tính toán khuyến mãi.
Database server (Data layer): lưu trữ sản phẩm và giao dịch.

Hãy mô tả:

Luồng xử lý khi khách hàng đặt một đơn hàng mới.
Ưu điểm của việc tách application server ra thành một tầng riêng thay vì để toàn bộ xử lý ở client hoặc database server.


BÀI GIẢI:
1. Luồng xử lý khi khách hàng đặt một đơn hàng mới
Kiến trúc hệ thống:
┌─────────────────────┐
│   CLIENT TIER       │
│   (UI Layer)        │
│  - Web Browser      │
│  - Mobile App       │
└──────────┬──────────┘
           │ HTTP/HTTPS Request
           ▼
┌─────────────────────┐
│ APPLICATION TIER    │
│ (Processing Layer)  │
│  - Business Logic   │
│  - Order Processing │
│  - Promotion Engine │
└──────────┬──────────┘
           │ SQL Query
           ▼
┌─────────────────────┐
│   DATA TIER         │
│   (Database Layer)  │
│  - Product DB       │
│  - Order DB         │
│  - Customer DB      │
└─────────────────────┘
Chi tiết các bước xử lý đơn hàng:
BƯỚC 1: Client Tier - Thu thập thông tin đơn hàng
Khách hàng thực hiện:
1. Đăng nhập vào website (email/password)
2. Duyệt sản phẩm, xem thông tin chi tiết
3. Thêm sản phẩm vào giỏ hàng (Product ID: 101, Quantity: 2)
4. Nhập thông tin giao hàng (địa chỉ, số điện thoại)
5. Chọn phương thức thanh toán (COD/Credit Card)
6. Click nút "Đặt hàng"
Action từ Client:

Browser thu thập dữ liệu form
Validate cơ bản phía client (required fields, email format)
Tạo HTTP POST request chứa thông tin đơn hàng
Gửi request đến Application Server endpoint: POST /api/orders

Request payload (JSON):
json{
  "customer_id": 12345,
  "items": [
    {"product_id": 101, "quantity": 2},
    {"product_id": 205, "quantity": 1}
  ],
  "shipping_address": {
    "street": "123 Nguyen Hue",
    "city": "Ho Chi Minh",
    "zip": "700000"
  },
  "payment_method": "credit_card",
  "promo_code": "SUMMER2024"
}
```

---

**BƯỚC 2: Application Tier - Xử lý nghiệp vụ**

Application Server nhận request và thực hiện chuỗi xử lý:

**2.1. Authentication & Authorization**
```
- Verify JWT token hoặc session ID
- Kiểm tra customer_id có khớp với user đang login không
- Check quyền đặt hàng (tài khoản có bị khóa không?)
```

**2.2. Validation nghiệp vụ**
```
Query Database để kiểm tra:
- Sản phẩm có tồn tại không?
  SQL: SELECT * FROM products WHERE product_id IN (101, 205)
  
- Số lượng tồn kho có đủ không?
  SQL: SELECT stock_quantity FROM inventory 
       WHERE product_id = 101 AND warehouse_id = 1
  
Kết quả: Product 101 có 50 items, đủ để bán 2 items
```

**2.3. Tính toán giá và khuyến mãi**
```
Business Logic thực thi:

a) Lấy giá sản phẩm từ database:
   - Product 101: 500,000 VND × 2 = 1,000,000 VND
   - Product 205: 300,000 VND × 1 = 300,000 VND
   - Subtotal = 1,300,000 VND

b) Xử lý mã khuyến mãi "SUMMER2024":
   - Query: SELECT * FROM promotions 
            WHERE code = 'SUMMER2024' 
            AND valid_from <= NOW() 
            AND valid_to >= NOW()
   - Kết quả: Giảm 10% cho đơn hàng trên 1 triệu
   - Discount = 1,300,000 × 10% = 130,000 VND

c) Tính phí vận chuyển:
   - Logic: Nếu subtotal > 1 triệu → Free ship
   - Shipping fee = 0 VND

d) Tổng cuối:
   Total = 1,300,000 - 130,000 + 0 = 1,170,000 VND
```

**2.4. Tạo đơn hàng trong database**
```
Application Server thực hiện transaction:

BEGIN TRANSACTION;

-- Tạo bản ghi đơn hàng
INSERT INTO orders (customer_id, order_date, status, total_amount)
VALUES (12345, NOW(), 'PENDING', 1170000);
-- Giả sử trả về order_id = 9876

-- Tạo chi tiết đơn hàng
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES 
  (9876, 101, 2, 500000),
  (9876, 205, 1, 300000);

-- Giảm tồn kho
UPDATE inventory 
SET stock_quantity = stock_quantity - 2
WHERE product_id = 101 AND warehouse_id = 1;

UPDATE inventory 
SET stock_quantity = stock_quantity - 1
WHERE product_id = 205 AND warehouse_id = 1;

-- Lưu thông tin giao hàng
INSERT INTO shipping_info (order_id, address, city, zip)
VALUES (9876, '123 Nguyen Hue', 'Ho Chi Minh', '700000');

-- Ghi log khuyến mãi đã sử dụng
INSERT INTO promotion_usage (order_id, promo_code, discount_amount)
VALUES (9876, 'SUMMER2024', 130000);

COMMIT;
```

**2.5. Xử lý thanh toán**
```
Nếu payment_method = "credit_card":
  - Call Payment Gateway API (Stripe/PayPal)
  - Request payment authorization
  - Nhận response: transaction_id, status
  
  UPDATE orders 
  SET payment_status = 'AUTHORIZED', 
      transaction_id = 'txn_abc123'
  WHERE order_id = 9876;
```

**2.6. Gửi thông báo**
```
Application Server trigger các events:
- Gửi email xác nhận đơn hàng cho khách hàng
- Gửi notification đến kho để chuẩn bị hàng
- Gửi thông báo SMS (nếu có)
- Push notification đến mobile app

BƯỚC 3: Application Server trả response về Client
jsonHTTP 201 Created
{
  "success": true,
  "order_id": 9876,
  "status": "PENDING",
  "total_amount": 1170000,
  "estimated_delivery": "2024-12-15",
  "message": "Đơn hàng đã được đặt thành công!"
}
```

---

**BƯỚC 4: Client Tier - Hiển thị kết quả**
```
Browser nhận response:
1. Parse JSON response
2. Hiển thị trang xác nhận đơn hàng
3. Show order_id: #9876
4. Display thông tin: số tiền, ngày giao hàng dự kiến
5. Redirect to order tracking page
```

---

**Sơ đồ luồng hoàn chỉnh:**
```
CLIENT                  APPLICATION SERVER              DATABASE
  │                              │                          │
  │  1. POST /api/orders        │                          │
  ├────────────────────────────>│                          │
  │                              │  2. Verify token         │
  │                              │                          │
  │                              │  3. Query products       │
  │                              ├─────────────────────────>│
  │                              │<─────────────────────────┤
  │                              │  4. Check inventory      │
  │                              ├─────────────────────────>│
  │                              │<─────────────────────────┤
  │                              │  5. Query promotions     │
  │                              ├─────────────────────────>│
  │                              │<─────────────────────────┤
  │                              │                          │
  │                              │  6. Calculate total      │
  │                              │     (Business Logic)     │
  │                              │                          │
  │                              │  7. BEGIN TRANSACTION    │
  │                              ├─────────────────────────>│
  │                              │  8. INSERT orders        │
  │                              ├─────────────────────────>│
  │                              │  9. INSERT order_items   │
  │                              ├─────────────────────────>│
  │                              │ 10. UPDATE inventory     │
  │                              ├─────────────────────────>│
  │                              │ 11. COMMIT               │
  │                              ├─────────────────────────>│
  │                              │<─────────────────────────┤
  │                              │                          │
  │                              │ 12. Call Payment API     │
  │                              │     (external service)   │
  │                              │                          │
  │                              │ 13. Send notifications   │
  │                              │                          │
  │  14. Response (order_id)    │                          │
  │<────────────────────────────┤                          │
  │                              │                          │
  │  15. Display confirmation   │                          │
```

---

#### **2. Ưu điểm của việc tách Application Server ra thành một tầng riêng**

**A. So sánh 3 kiến trúc:**

**Mô hình 1: Two-tier (Client xử lý logic)**
```
┌─────────────┐          ┌─────────────┐
│   Client    │          │  Database   │
│ (Fat Client)│─────────>│   Server    │
│ + UI        │          │             │
│ + Logic     │<─────────│             │
│ + Validation│          │             │
└─────────────┘          └─────────────┘
```

**Mô hình 2: Two-tier (Database xử lý logic)**
```
┌─────────────┐          ┌─────────────┐
│   Client    │          │  Database   │
│ (Thin Client)─────────>│   Server    │
│ + UI only   │          │ + Logic     │
│             │<─────────│ + Stored Proc│
└─────────────┘          └─────────────┘
```

**Mô hình 3: Three-tier (Application Server riêng)**
```
┌────────┐    ┌────────────────┐    ┌──────────┐
│ Client │───>│ Application    │───>│ Database │
│  + UI  │<───│    Server      │<───│  Server  │
└────────┘    │ + Logic        │    └──────────┘
              │ + Validation   │
              │ + API          │
              └────────────────┘
```

---

**B. Ưu điểm chi tiết của kiến trúc 3 tầng:**

**2.1. Tách biệt trách nhiệm (Separation of Concerns)**

✅ **Client chỉ lo presentation:**
- Render UI, handle user interactions
- Không chứa business logic → Code đơn giản hơn
- Dễ phát triển responsive design, mobile-friendly

✅ **Application Server lo business logic:**
- Xử lý nghiệp vụ tập trung
- Validation, tính toán, workflow
- Không lo về UI rendering

✅ **Database chỉ lo lưu trữ:**
- Tối ưu cho query performance
- Không chứa business logic phức tạp
- Dễ backup, replicate

**Ví dụ thực tế:**
```
Khi thay đổi quy tắc khuyến mãi từ "giảm 10%" thành "giảm 15%":
- Three-tier: Chỉ sửa code ở Application Server
- Two-tier (logic ở client): Phải update app trên hàng triệu thiết bị!

2.2. Khả năng bảo mật cao hơn
✅ Business logic không lộ ra client:

Client không thấy được cách tính giá, thuật toán khuyến mãi
Hacker không thể reverse engineer app để xem logic
Giảm nguy cơ bị bypass validation

❌ Nếu logic ở client (Two-tier):
javascript// Code JavaScript trên browser (ai cũng đọc được!)
function calculateDiscount(total) {
  if (total > 1000000) {
    return total * 0.1; // Giảm 10%
  }
  return 0;
}
// → Hacker có thể modify code để luôn được giảm giá
```

✅ **Three-tier (Logic ở server):**
```
Client chỉ gửi: promo_code = "SUMMER2024"
Server tự tính toán (client không biết logic)
→ Không thể giả mạo
```

✅ **Credential database được bảo vệ:**
- Client không kết nối trực tiếp đến database
- Database chỉ accept connection từ Application Server
- Thêm một lớp firewall giữa client và data

---

**2.3. Khả năng mở rộng linh hoạt (Scalability)**

✅ **Scale từng tầng độc lập:**
```
Load thấp:
[1 Web Server] ──> [1 App Server] ──> [1 Database]

Load trung bình:
[2 Web Servers] ──> [3 App Servers] ──> [1 Database]
     └─ Load Balancer ─┴─────┘

Load cao (Black Friday):
[5 Web Servers] ──> [20 App Servers] ──> [1 Master DB]
     └─ Load Balancer ─┴─────┘              └─> [3 Read Replicas]
```

**Ví dụ thực tế:**
- Black Friday: Traffic tăng 10x → Chỉ cần thêm Application Server instances
- Không cần thêm database server (vì DB không phải bottleneck)
- Chi phí tiết kiệm hơn việc scale cả hệ thống

❌ **Two-tier không làm được điều này:**
- Phải scale cả client + database cùng lúc
- Không linh hoạt, tốn kém

---

**2.4. Dễ bảo trì và nâng cấp (Maintainability)**

✅ **Sửa code không ảnh hưởng client:**

**Ví dụ:** Thêm tính năng "Loyalty Points"
```
Three-tier:
1. Sửa Application Server: thêm logic tính điểm
2. Client tự động nhận được tính năng mới qua API
3. Không cần update app trên mobile store

Two-tier:
1. Phải release app version mới
2. Đợi user update (có thể mất vài tháng)
3. Phải maintain nhiều version đồng thời
```

✅ **Rollback dễ dàng:**
```
Nếu phát hiện bug sau khi deploy:
- Three-tier: Rollback Application Server (5 phút)
- Two-tier: Không thể rollback app đã cài trên máy user
```

✅ **A/B Testing:**
```
Application Server có thể:
- Route 10% traffic đến logic mới (test)
- Route 90% traffic đến logic cũ (stable)
→ Thu thập metrics rồi quyết định
```

---

**2.5. Hiệu năng tốt hơn (Performance)**

✅ **Caching hiệu quả:**
```
Application Server cache:
- Product catalog (1 giờ)
- Promotion rules (5 phút)
- User sessions (memory)

→ Giảm 80% queries đến database
→ Response time từ 500ms xuống 50ms
```

❌ **Two-tier (logic ở client):**
- Mỗi client phải fetch data riêng
- Không share cache giữa các users
- Database chịu tải nặng hơn

✅ **Connection pooling:**
```
Application Server duy trì:
- 50 connection pool đến database
- Tái sử dụng connections
- Giảm overhead của việc tạo connection mới

Two-tier:
- Mỗi client tạo connection riêng
- 10,000 users = 10,000 connections
- Database crash!
```

---

**2.6. Hỗ trợ nhiều loại client (Multi-platform)**

✅ **Một API phục vụ nhiều client:**
```
                  ┌──> Web Browser
                  │
Application ─────┼──> Mobile iOS App
   Server         │
(REST API)       ┼──> Mobile Android App
                  │
                  ┼──> Desktop App
                  │
                  └──> IoT Device / Smart TV
```

**Lợi ích:**
- Business logic viết 1 lần, dùng cho tất cả platforms
- Đảm bảo consistency (tính giá giống nhau trên mọi nền tảng)
- Khi sửa bug, tất cả platforms đều được fix

❌ **Two-tier:**
- Phải implement logic riêng cho iOS, Android, Web
- Khó đảm bảo consistency
- Sửa bug phải sửa 3 lần

---

**2.7. Tích hợp dễ dàng (Integration)**

✅ **Application Server là integration hub:**
```
                    ┌──> Payment Gateway (Stripe)
                    │
Application ───────┼──> Email Service (SendGrid)
  Server            │
                    ┼──> SMS Service (Twilio)
                    │
                    ┼──> Analytics (Google Analytics)
                    │
                    └──> Inventory System (ERP)
```

**Lợi ích:**
- Third-party integrations tập trung
- Client không cần biết chi tiết các dịch vụ bên ngoài
- Dễ thay đổi provider (từ Stripe sang PayPal)

---

**2.8. Monitoring và Logging tập trung**

✅ **Application Server làm central logging:**
```
Mọi request đi qua App Server:
- Log request/response
- Track performance metrics
- Detect anomalies (fraud detection)
- Generate analytics reports

Dashboard hiển thị:
- Tỷ lệ thành công/thất bại đơn hàng
- Average response time
- Most popular products
- Peak traffic hours
❌ Two-tier:

Log phân tán trên nhiều clients
Khó tổng hợp và phân tích


2.9. Kiểm thử dễ dàng hơn (Testability)
✅ Unit testing business logic:
python# Test trên Application Server
def test_calculate_discount():
    order = Order(total=1500000, promo_code="SUMMER2024")
    discount = calculate_discount(order)
    assert discount == 150000  # 10%

# Không cần chạy UI để test logic
# CI/CD tự động test mỗi khi commit code
```

✅ **Integration testing:**
```
Test Application Server với mock database
→ Nhanh, không phụ thuộc infrastructure
```

---

**2.10. Business Continuity**

✅ **Failover và High Availability:**
```
Load Balancer
    │
    ├──> App Server 1 (Active)
    ├──> App Server 2 (Active)
    ├──> App Server 3 (Standby)
    └──> App Server 4 (Standby)

Nếu App Server 1 crash:
→ Load Balancer tự động route traffic sang Server 2
→ Users không bị ảnh hưởng
❌ Two-tier:

Client kết nối trực tiếp database
Database down = toàn bộ hệ thống down
Không có lớp abstraction để failover


Tóm tắt so sánh:
Tiêu chíTwo-tier (Logic ở Client)Two-tier (Logic ở DB)Three-tierBảo mật❌ Kém (logic lộ ra)⚠️ Trung bình✅ TốtScalability❌ Khó scale⚠️ Chỉ scale DB✅ Scale độc lậpMaintainability❌ Khó (cần update app)⚠️ Stored proc khó maintain✅ DễPerformance⚠️ Cache riêng lẻ❌ DB overload✅ Tốt (shared cache)Multi-platform❌ Duplicate code❌ Duplicate code✅ Một API cho tất cảTesting❌ Khó (cần UI)⚠️ Phức tạp✅ Dễ (unit test)Deployment❌ Phải update client⚠️ Sửa DB rủi ro✅ Deploy server nhanh

Kết luận:
Kiến trúc 3 tầng với Application Server riêng biệt mang lại nhiều lợi ích vượt trội cho hệ thống bán hàng trực tuyến:

Bảo mật tốt hơn: Business logic được bảo vệ
Linh hoạt hơn: Scale và maintain từng tầng độc lập
Hiệu năng cao hơn: Caching, connection pooling tập trung
Phát triển nhanh hơn: Một API cho nhiều platforms

Đây là lý do tại sao hầu hết các hệ thống enterprise hiện đại đều áp dụng kiến trúc 3 tầng hoặc nhiều hơn (microservices).