# Câu 2: Kiến trúc 3 Tầng (Three-Tier Architecture)

> **Chương:** 2 - Kiến trúc Hệ thống Phân tán  
> **Độ khó:** ⭐⭐⭐ (Trung bình)  
> **Thời gian đọc:** ~20 phút

---

## 📋 Mục lục

- [Đề bài](#đề-bài)
- [Phần 1: Mô tả kiến trúc 3 tầng](#phần-1-mô-tả-kiến-trúc-3-tầng)
- [Phần 2: Luồng xử lý đơn hàng](#phần-2-luồng-xử-lý-đơn-hàng)
- [Phần 3: Ưu điểm của Application Server riêng](#phần-3-ưu-điểm-của-application-server-riêng)
- [Phần 4: So sánh với kiến trúc 2 tầng](#phần-4-so-sánh-với-kiến-trúc-2-tầng)
- [Tóm tắt](#tóm-tắt)

---

## 📋 Đề bài

Một hệ thống thương mại điện tử được thiết kế theo kiến trúc 3 tầng (three-tiered architecture):

- **Client (presentation tier)**: giao diện web/mobile cho khách hàng
- **Application server (processing tier)**: xử lý đơn hàng, tính toán khuyến mãi
- **Database server (data tier)**: lưu trữ sản phẩm và giao dịch

**Yêu cầu:**

1. Mô tả luồng xử lý khi khách hàng đặt một đơn hàng, từ client → application server → database
2. Giải thích tại sao việc tách riêng **application server** mang lại lợi ích về:
   - **Bảo mật** (security)
   - **Khả năng mở rộng** (scalability)  
   - **Khả năng bảo trì** (maintainability)

---

## 💡 Bài giải

### Phần 1: Mô tả kiến trúc 3 tầng

#### A. Tổng quan kiến trúc
```
┌─────────────────────────────────────────────────┐
│         TIER 1: CLIENT (Presentation)           │
│                                                 │
│  ┌──────────────┐  ┌──────────────┐           │
│  │ Web Browser  │  │ Mobile App   │           │
│  │   (React)    │  │  (Flutter)   │           │
│  └──────────────┘  └──────────────┘           │
│                                                 │
│  Responsibilities:                              │
│  - UI/UX rendering                              │
│  - User input collection                        │
│  - Display data from server                     │
│  - Client-side validation                       │
└─────────────────┬───────────────────────────────┘
                  │ HTTPS / REST API
┌─────────────────▼───────────────────────────────┐
│       TIER 2: APPLICATION SERVER (Logic)        │
│                                                 │
│  ┌────────────────────────────────────────────┐│
│  │  Business Logic Layer                      ││
│  │  - Order processing                        ││
│  │  - Promotion calculation                   ││
│  │  - Inventory validation                    ││
│  │  - Payment processing                      ││
│  │  - Notification service                    ││
│  └────────────────────────────────────────────┘│
│                                                 │
│  Technologies:                                  │
│  - Java Spring Boot / Node.js / Python Django  │
│  - Load Balancer (HAProxy, Nginx)             │
│  - Application servers (clustered)             │
└─────────────────┬───────────────────────────────┘
                  │ JDBC / SQL
┌─────────────────▼───────────────────────────────┐
│        TIER 3: DATABASE SERVER (Data)           │
│                                                 │
│  ┌────────────────────────────────────────────┐│
│  │  Relational Database (PostgreSQL/MySQL)    ││
│  │                                            ││
│  │  Tables:                                   ││
│  │  - customers                               ││
│  │  - products                                ││
│  │  - orders                                  ││
│  │  - order_items                             ││
│  │  - inventory                               ││
│  │  - transactions                            ││
│  └────────────────────────────────────────────┘│
│                                                 │
│  Features:                                      │
│  - ACID transactions                            │
│  - Replication (master-slave)                  │
│  - Backup and recovery                         │
└─────────────────────────────────────────────────┘
```

---

### Phần 2: Luồng xử lý đơn hàng

#### A. Chi tiết từng bước
```
BƯỚC 1: Client → Request
═══════════════════════════════════════════════

┌─────────────────────────────────────────┐
│  User Action (Web/Mobile)               │
│                                         │
│  1. Browse products                     │
│  2. Add to cart: Product A (Qty: 2)    │
│  3. Add to cart: Product B (Qty: 1)    │
│  4. Click "Checkout"                    │
│  5. Enter shipping info                 │
│  6. Select payment method               │
│  7. Click "Place Order"                 │
└─────────────────┬───────────────────────┘
                  │
                  ▼
         HTTP POST Request
         
POST /api/orders HTTP/1.1
Host: api.ecommerce.com
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "customer_id": "C12345",
  "items": [
    {
      "product_id": "P101",
      "quantity": 2,
      "price": 99.99
    },
    {
      "product_id": "P205",
      "quantity": 1,
      "price": 149.99
    }
  ],
  "shipping_address": {
    "street": "123 Nguyen Hue",
    "city": "Ho Chi Minh",
    "country": "Vietnam"
  },
  "payment_method": "credit_card",
  "promo_code": "SUMMER2024"
}
```
```
BƯỚC 2: Application Server → Processing
═══════════════════════════════════════════════

┌────────────────────────────────────────────────┐
│  APPLICATION SERVER (Business Logic)           │
│                                                │
│  Step 2.1: Authentication & Authorization      │
│  ├─ Verify JWT token                          │
│  ├─ Check user session                        │
│  └─ Validate customer_id                      │
│                                                │
│  Step 2.2: Input Validation                    │
│  ├─ Check required fields                     │
│  ├─ Validate product IDs exist                │
│  ├─ Validate quantities (> 0)                 │
│  └─ Validate shipping address format          │
│                                                │
│  Step 2.3: Business Logic Processing          │
│  ┌──────────────────────────────────────┐    │
│  │ a) Check Inventory Availability      │    │
│  │    Query: SELECT stock FROM products │    │
│  │    WHERE product_id IN ('P101','P205')│   │
│  │                                       │    │
│  │    Result:                            │    │
│  │    - P101: stock = 50 (sufficient)   │    │
│  │    - P205: stock = 5 (sufficient)    │    │
│  └──────────────────────────────────────┘    │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │ b) Calculate Promotion               │    │
│  │    Subtotal: 2×99.99 + 1×149.99     │    │
│  │            = 349.97                  │    │
│  │                                       │    │
│  │    Promo Code: SUMMER2024            │    │
│  │    Discount: 10%                     │    │
│  │    Savings: 34.99                    │    │
│  │                                       │    │
│  │    Shipping: 15.00                   │    │
│  │    Tax (10%): 33.00                  │    │
│  │                                       │    │
│  │    TOTAL: 363.98                     │    │
│  └──────────────────────────────────────┘    │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │ c) Create Transaction (Database)     │    │
│  │                                       │    │
│  │    BEGIN TRANSACTION;                │    │
│  │                                       │    │
│  │    -- Insert order                   │    │
│  │    INSERT INTO orders (              │    │
│  │      order_id, customer_id,          │    │
│  │      total_amount, status,           │    │
│  │      created_at                      │    │
│  │    ) VALUES (                        │    │
│  │      'ORD-2024-12345',               │    │
│  │      'C12345',                       │    │
│  │      363.98,                         │    │
│  │      'PENDING',                      │    │
│  │      NOW()                           │    │
│  │    );                                │    │
│  │                                       │    │
│  │    -- Insert order items             │    │
│  │    INSERT INTO order_items ...       │    │
│  │                                       │    │
│  │    -- Update inventory               │    │
│  │    UPDATE products                   │    │
│  │    SET stock = stock - 2             │    │
│  │    WHERE product_id = 'P101';        │    │
│  │                                       │    │
│  │    UPDATE products                   │    │
│  │    SET stock = stock - 1             │    │
│  │    WHERE product_id = 'P205';        │    │
│  │                                       │    │
│  │    COMMIT;                           │    │
│  └──────────────────────────────────────┘    │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │ d) Payment Processing                │    │
│  │    Call Payment Gateway API          │    │
│  │    POST https://payment.stripe.com   │    │
│  │    {                                 │    │
│  │      amount: 363.98,                 │    │
│  │      currency: "USD",                │    │
│  │      card_token: "tok_visa"          │    │
│  │    }                                 │    │
│  │                                       │    │
│  │    Response: SUCCESS                 │    │
│  │    Transaction ID: txn_abc123        │    │
│  └──────────────────────────────────────┘    │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │ e) Post-Processing                   │    │
│  │    - Update order status: CONFIRMED  │    │
│  │    - Send confirmation email         │    │
│  │    - Send SMS notification           │    │
│  │    - Trigger warehouse fulfillment   │    │
│  └──────────────────────────────────────┘    │
└────────────────────────────────────────────────┘
```
```
BƯỚC 3: Application Server → Client Response
═══════════════════════════════════════════════

HTTP/1.1 201 Created
Content-Type: application/json

{
  "status": "success",
  "order_id": "ORD-2024-12345",
  "order_number": "#12345",
  "total_amount": 363.98,
  "estimated_delivery": "2024-12-18",
  "message": "Order placed successfully!",
  "tracking_url": "https://track.ecommerce.com/ORD-2024-12345"
}

┌─────────────────────────────────────────┐
│  Client Display                         │
│                                         │
│  ✅ Order Confirmed!                    │
│                                         │
│  Order Number: #12345                   │
│  Total: $363.98                         │
│  Estimated Delivery: Dec 18, 2024      │
│                                         │
│  [Track Order] [View Receipt]          │
└─────────────────────────────────────────┘
```

#### B. Code Implementation (Java Spring Boot)
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request,
            @AuthenticationPrincipal User user) {
        
        try {
            // STEP 1: Validate input
            if (!orderService.validateOrder(request)) {
                return ResponseEntity.badRequest()
                    .body(new OrderResponse("Invalid order data"));
            }
            
            // STEP 2: Process order (business logic)
            Order order = orderService.processOrder(request, user);
            
            // STEP 3: Return response
            OrderResponse response = new OrderResponse(
                "success",
                order.getOrderId(),
                order.getTotalAmount(),
                "Order placed successfully!"
            );
            
            return ResponseEntity.status(HttpStatus.CREATED)
                .body(response);
                
        } catch (InsufficientStockException e) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(new OrderResponse("Insufficient stock"));
                
        } catch (PaymentFailedException e) {
            return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED)
                .body(new OrderResponse("Payment failed"));
                
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new OrderResponse("Order processing failed"));
        }
    }
}

@Service
@Transactional
public class OrderService {
    
    @Autowired
    private ProductRepository productRepo;
    
    @Autowired
    private OrderRepository orderRepo;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    public Order processOrder(OrderRequest request, User user) {
        
        // 1. Check inventory
        for (OrderItem item : request.getItems()) {
            Product product = productRepo.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException());
            
            if (product.getStock() < item.getQuantity()) {
                throw new InsufficientStockException(product.getName());
            }
        }
        
        // 2. Calculate totals
        BigDecimal subtotal = calculateSubtotal(request.getItems());
        BigDecimal discount = calculateDiscount(request.getPromoCode(), subtotal);
        BigDecimal shipping = calculateShipping(request.getShippingAddress());
        BigDecimal tax = calculateTax(subtotal.subtract(discount));
        BigDecimal total = subtotal.subtract(discount).add(shipping).add(tax);
        
        // 3. Create order (database transaction)
        Order order = new Order();
        order.setCustomerId(user.getId());
        order.setTotalAmount(total);
        order.setStatus(OrderStatus.PENDING);
        order.setCreatedAt(LocalDateTime.now());
        
        // Save order
        order = orderRepo.save(order);
        
        // Save order items
        for (OrderItem item : request.getItems()) {
            item.setOrderId(order.getId());
            orderRepo.saveOrderItem(item);
        }
        
        // 4. Update inventory
        for (OrderItem item : request.getItems()) {
            productRepo.decrementStock(
                item.getProductId(),
                item.getQuantity()
            );
        }
        
        // 5. Process payment
        PaymentResult paymentResult = paymentService.charge(
            request.getPaymentMethod(),
            total,
            order.getOrderId()
        );
        
        if (!paymentResult.isSuccessful()) {
            throw new PaymentFailedException();
        }
        
        // 6. Update order status
        order.setStatus(OrderStatus.CONFIRMED);
        order.setPaymentId(paymentResult.getTransactionId());
        orderRepo.save(order);
        
        // 7. Send notifications (async)
        notificationService.sendOrderConfirmation(user.getEmail(), order);
        notificationService.sendSMS(user.getPhone(), order);
        
        // 8. Trigger fulfillment
        warehouseService.createFulfillmentOrder(order);
        
        return order;
    }
    
    private BigDecimal calculateDiscount(String promoCode, BigDecimal subtotal) {
        if (promoCode == null || promoCode.isEmpty()) {
            return BigDecimal.ZERO;
        }
        
        PromoCode promo = promoCodeRepo.findByCode(promoCode)
            .orElse(null);
        
        if (promo == null || !promo.isValid()) {
            return BigDecimal.ZERO;
        }
        
        // Apply discount (e.g., 10% = 0.10)
        return subtotal.multiply(promo.getDiscountRate());
    }
}
```

---

### Phần 3: Ưu điểm của Application Server riêng

#### A. Bảo mật (Security)

**1. Separation of Concerns**
```
┌────────────────────────────────────────────┐
│  CLIENT (Untrusted)                        │
│  - User can inspect network traffic        │
│  - User can modify client-side code        │
│  - User can bypass client validation       │
└────────────────┬───────────────────────────┘
                 │ HTTPS only
                 │ No direct DB access ✅
┌────────────────▼───────────────────────────┐
│  APPLICATION SERVER (Trusted)              │
│  - Business logic hidden from client       │
│  - Server-side validation (cannot bypass)  │
│  - Authentication & Authorization          │
│  - Rate limiting                           │
│  - Input sanitization                      │
└────────────────┬───────────────────────────┘
                 │ Internal network
                 │ Firewall protected
┌────────────────▼───────────────────────────┐
│  DATABASE (Most Secure)                    │
│  - No direct internet access               │
│  - Only app server can connect             │
│  - Credentials stored in app server only   │
└────────────────────────────────────────────┘

Security benefits:
✅ Database credentials NEVER exposed to client
✅ Business logic cannot be reverse-engineered
✅ Server-side validation prevents tampering
✅ Centralized authentication & authorization
```

**2. Example: SQL Injection Prevention**

**❌ BAD: Client connects directly to database**
```javascript
// Client-side code (INSECURE!)
const userId = getUserInput(); // User enters: "1 OR 1=1"

// Direct SQL query from client
const query = `SELECT * FROM users WHERE id = ${userId}`;
// Result: SELECT * FROM users WHERE id = 1 OR 1=1
// → Returns ALL users! SQL injection attack! ❌
```

**✅ GOOD: Application server mediates**
```javascript
// Client sends request
POST /api/users/1 HTTP/1.1

// Application server (SECURE)
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    // Parameterized query (safe from SQL injection)
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException());
}

// Database receives:
// SELECT * FROM users WHERE id = ? [Parameter: 1]
// Even if user sends "1 OR 1=1", it's treated as literal string ✅
```

**3. Credential Protection**
```
❌ TWO-TIER (Client → Database):
┌─────────────────────────────────────┐
│  Client Application                 │
│                                     │
│  Database credentials in config:    │
│  DB_HOST=db.company.com             │
│  DB_USER=admin                      │
│  DB_PASS=secret123                  │
│                                     │
│  Risk: Anyone can decompile app     │
│  and steal credentials! ❌          │
└─────────────────────────────────────┘

✅ THREE-TIER (Client → App Server → Database):
┌─────────────────────────────────────┐
│  Client                             │
│  - Only knows API endpoint          │
│  - No database credentials ✅       │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│  Application Server                 │
│  - Credentials in env variables     │
│  - Never sent to client ✅          │
│  - Can rotate without client update │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│  Database                           │
│  - Only accepts app server IP ✅    │
└─────────────────────────────────────┘
```

---

#### B. Khả năng mở rộng (Scalability)

**1. Independent Scaling**
```
Scenario: Black Friday sale
─────────────────────────────────────────

Traffic: 1,000 requests/second (100x normal)

┌──────────────────────────────────────────┐
│  TIER 1: Clients (1M concurrent users)   │
└──────────────────┬───────────────────────┘
                   │
         ┌─────────▼─────────┐
         │  Load Balancer    │
         └─────────┬─────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ▼              ▼              ▼
┌────────┐    ┌────────┐    ┌────────┐
│ App    │    │ App    │    │ App    │  ... × 20
│Server 1│    │Server 2│    │Server 3│
└───┬────┘    └───┬────┘    └───┬────┘
    │             │             │
    └─────────────┼─────────────┘
                  │
         ┌────────▼────────┐
         │  Database       │  × 1
         │  (Master)       │
         └─────────────────┘

Scaling strategy:
✅ Scale OUT application tier: 1 → 20 servers
✅ Database remains: 1 server (sufficient)

Cost:
- App servers: $100/month × 20 = $2,000
- Database: $500/month × 1 = $500
- Total: $2,500/month

Alternative (2-tier):
- Would need to scale database too (expensive!)
- Database scaling is much more complex
```

**2. Caching Layer**
```
┌─────────────────────────────────────────┐
│  APPLICATION SERVER TIER                │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Redis Cache (Shared)           │   │
│  │  - Product catalog              │   │
│  │  - User sessions                │   │
│  │  - Promotion rules              │   │
│  └─────────────────────────────────┘   │
│           ▲                             │
│           │ 90% requests hit cache ✅   │
│  ┌────────┴────────┬───────────────┐   │
│  │                 │               │   │
│  ▼                 ▼               ▼   │
│ App Server 1   App Server 2   App Server 3
└─────────────────────────────────────────┘
          │ Only 10% hit database
          ▼
┌─────────────────────────────────────────┐
│  DATABASE                               │
│  Load: 10% of original ✅               │
└─────────────────────────────────────────┘

Performance improvement:
- Cache hit: <1ms response time ✅
- Database query: 50-100ms ⚠️
- 90% requests 50x faster!
```

**3. Horizontal vs Vertical Scaling**
```
╔══════════════════════════════════════════════╗
║         SCALING COMPARISON                   ║
╠══════════════════════════════════════════════╣
║                                              ║
║  Tier        │ Scaling    │ Cost      │ Max ║
║              │ Strategy   │           │     ║
║ ═════════════╪════════════╪═══════════╪═════║
║                                              ║
║  Client      │ Infinite   │ Free      │ ∞   ║
║              │ (users)    │           │     ║
║              │            │           │     ║
║  App Server  │ Horizontal │ $100/each │ 100+║
║              │ (add more) │ Linear ✅ │     ║
║              │            │           │     ║
║  Database    │ Vertical   │ $500-5K   │ 1-3 ║
║              │ (bigger)   │ Exp! ⚠️   │     ║
║              │            │           │     ║
╚══════════════════════════════════════════════╝

Example cost to handle 10K req/s:

Option A: Scale app tier only
- 20 app servers: $2,000/month
- 1 database: $500/month
- Total: $2,500/month ✅

Option B: Scale database (2-tier)
- 1 massive database: $8,000/month
- Total: $8,000/month ❌

Savings: 68% ✅
```

---

#### C. Khả năng bảo trì (Maintainability)

**1. Independent Deployment**
```
Scenario: Update promotion algorithm

┌──────────────────────────────────────────┐
│  APPLICATION SERVER CODE                 │
│                                          │
│  Old version (v1.0):                     │
│  discount = subtotal * 0.10              │
│                                          │
│  New version (v1.1):                     │
│  discount = calculateTieredDiscount()    │
│  - 0-$100: 5%                           │
│  - $100-$500: 10%                       │
│  - $500+: 15%                           │
└──────────────────────────────────────────┘

Deployment (3-tier):
1. Update app server code
2. Deploy to staging
3. Test
4. Rolling deployment to production
5. Done! ✅

Impact:
✅ Clients: NO CHANGE (still use same API)
✅ Database: NO CHANGE (same schema)
✅ Zero downtime deployment

Deployment (2-tier):
1. Update client app
2. Update database stored procedures
3. Deploy new app to ALL users
4. Wait for users to update app ⚠️

Impact:
❌ Must update mobile app (App Store review)
❌ Users must download new version
❌ Old app versions broken
❌ 1-2 weeks deployment time
```

**2. A/B Testing**
```
A/B Test: New checkout flow

┌─────────────────────────────────────────┐
│  LOAD BALANCER                          │
│  - 50% traffic → App Server A (old)    │
│  - 50% traffic → App Server B (new)    │
└─────────────────────────────────────────┘

┌─────────────────┐  ┌─────────────────┐
│ App Server A    │  │ App Server B    │
│ (v1.0 - old)    │  │ (v1.1 - new)    │
│                 │  │                 │
│ - Single page   │  │ - Multi-step    │
│ - All fields    │  │ - Progressive   │
└─────────────────┘  └─────────────────┘
        │                     │
        └──────────┬──────────┘
                   │
         ┌─────────▼─────────┐
         │  Same Database    │
         └───────────────────┘

Monitor:
- Conversion rate A: 3.2%
- Conversion rate B: 4.5% ✅

Decision:
- B is 40% better!
- Deploy B to 100%
- No client update needed! ✅

With 2-tier:
- Cannot A/B test easily ❌
- All clients must use same version
```

**3. Bug Fix Example**
```
Bug discovered: Promotion code applied twice

Timeline (3-tier):
─────────────────────────────────────────
10:00 AM - Bug reported
10:15 AM - Fix identified in OrderService
10:30 AM - Code committed, tested
10:45 AM - Deployed to production
11:00 AM - Bug resolved ✅

Total: 1 hour ✅

Timeline (2-tier):
─────────────────────────────────────────
10:00 AM - Bug reported
10:15 AM - Fix client code
10:30 AM - Submit to App Store
Day 2    - Apple review (24-48 hours)
Day 3    - App approved, released
Week 2   - 80% users updated ⚠️

Total: 1-2 weeks ❌
20% users still affected!
```

**4. Team Structure**
```
3-Tier Organization:

┌────────────────────────────────────┐
│  Frontend Team (3 developers)     │
│  - React/Flutter specialists       │
│  - Focus on UX/UI                  │
│  - Deploy independently            │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│  Backend Team (5 developers)      │
│  - Java/Node.js specialists        │
│  - Business logic experts          │
│  - Deploy independently ✅         │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│  DBA Team (2 specialists)          │
│  - Database optimization           │
│  - Schema changes                  │
│  - Backup/recovery                 │
└────────────────────────────────────┘

Benefits:
✅ Separation of concerns
✅ Parallel development
✅ Specialized expertise
✅ No dependencies between teams
```

---

### Phần 4: So sánh với kiến trúc 2 tầng
```
╔═══════════════════════════════════════════════════════╗
║          2-TIER vs 3-TIER COMPARISON                  ║
╠═══════════════════════════════════════════════════════╣
║                                                       ║
║  Criteria        │ 2-Tier        │ 3-Tier            ║
║                  │ (Client-DB)   │ (Client-App-DB)   ║
║ ═════════════════╪═══════════════╪═══════════════════║
║                                                       ║
║  Security        │ Low ❌        │ High ✅           ║
║                  │ DB exposed    │ DB protected      ║
║                                                       ║
║  Scalability     │ Limited ⚠️    │ Excellent ✅      ║
║                  │ Scale DB      │ Scale app tier    ║
║                                                       ║
║  Deployment      │ Slow ❌       │ Fast ✅           ║
║                  │ 1-2 weeks     │ Minutes           ║
║                                                       ║
║  Maintenance     │ Difficult ❌  │ Easy ✅           ║
║                  │ Update all    │ Update server     ║
║                                                       ║
║  Cost (high load)│ Expensive ⚠️  │ Moderate ✅       ║
║                  │ Big DB        │ Many app servers  ║
║                                                       ║
║  Development     │ Complex ❌    │ Cleaner ✅        ║
║                  │ Mixed logic   │ Separated         ║
║                                                       ║
║  Testing         │ Harder ❌     │ Easier ✅         ║
║                  │ Need DB       │ Mock app server   ║
║                                                       ║
║  Best for        │ Small apps    │ Enterprise ✅     ║
║                  │ <100 users    │ Production        ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

---

## 📊 Tóm tắt

### Key Points

- ✅ **3-tier architecture**: Client → App Server → Database
- ✅ **Separation of concerns**: Each tier has specific responsibility
- ✅ **Security**: Database credentials never exposed to client
- ✅ **Scalability**: Scale app tier independently (cheap)
- ✅ **Maintainability**: Deploy updates without client changes
- ✅ **Team productivity**: Parallel development, specialized teams

### Luồng xử lý đơn hàng

1. **Client** → Send order request (HTTPS)
2. **App Server** → Validate, calculate, create transaction
3. **Database** → Store order, update inventory (ACID)
4. **App Server** → Process payment, send notifications
5. **Client** ← Return success response

### Lợi ích chính

| Khía cạnh | Lợi ích | Impact |
|-----------|---------|--------|
| **Security** | Credentials protected | ✅ Prevent data breach |
| **Scalability** | Horizontal scaling | ✅ 10x capacity at 3x cost |
| **Maintainability** | Independent deployment | ✅ Bug fix in 1 hour vs 2 weeks |
| **Performance** | Caching layer possible | ✅ 90% requests <1ms |
| **Development** | Team specialization | ✅ 2x faster feature delivery |

---

## 🔗 Tài liệu tham khảo

### Sách
- **Patterns of Enterprise Application Architecture** - Martin Fowler
- **Building Scalable Web Sites** - Cal Henderson
- **Web Scalability for Startup Engineers** - Artur Ejsmont

### Articles
- "Three-Tier Architecture" - Microsoft Azure Docs
- "Scaling Web Applications" - AWS Architecture Center

---

## 🧭 Navigation

**[⬅️ Câu 1: Wrapper & Message Broker](./cau-1-wrapper-message-broker.md)** | **[📚 Quay lại Chương 2](./README.md)** | **[➡️ Câu 3: Chord DHT](./cau-3-chord-dht.md)**

---

*Cập nhật lần cuối: 11/12/2025*