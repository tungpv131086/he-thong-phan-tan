CHƯƠNG 2 - CÂU 1
Đề bài:
Một công ty đang sử dụng một hệ thống quản lý kho (Warehouse Management System – WMS) cũ, trong đó giao diện API không tương thích với hệ thống thương mại điện tử mới của họ. Hãy áp dụng khái niệm wrapper để đề xuất cách tích hợp hai hệ thống này. Mô tả cách wrapper giúp giải quyết vấn đề tương thích giao diện và so sánh chi phí phát triển nếu sử dụng O(N²) wrappers trực tiếp giữa các hệ thống so với giải pháp message broker.

BÀI GIẢI:
1. Khái niệm Wrapper và vai trò trong tích hợp hệ thống
Wrapper là một lớp trung gian (middleware component) đóng vai trò là bộ chuyển đổi giao diện (interface adapter), cho phép hai hệ thống có API không tương thích có thể giao tiếp với nhau mà không cần sửa đổi code gốc của các hệ thống đó.
Trong trường hợp này, wrapper sẽ:

Nhận yêu cầu từ hệ thống thương mại điện tử mới
Chuyển đổi định dạng dữ liệu và protocol
Gọi API của WMS cũ với format phù hợp
Nhận kết quả từ WMS và chuyển đổi ngược về format mà hệ thống mới hiểu được

2. Đề xuất giải pháp tích hợp sử dụng Wrapper
Kiến trúc đề xuất:
Hệ thống E-commerce mới  →  [Wrapper Layer]  →  WMS cũ
         (REST API)          (Adapter/Translator)    (SOAP/Legacy API)
Các thành phần của Wrapper:
a) Protocol Adapter: Chuyển đổi giữa các giao thức khác nhau (ví dụ: REST → SOAP)
b) Data Transformer: Chuyển đổi cấu trúc dữ liệu:

Input: JSON format từ e-commerce
Output: XML format cho WMS cũ

c) Method Mapper: Ánh xạ các phương thức API:

POST /orders (e-commerce) → CreateWarehouseOrder() (WMS)
GET /inventory/:id → CheckStockLevel(productId)

Ví dụ cụ thể:
E-commerce gửi yêu cầu kiểm tra tồn kho:
GET /api/inventory?product_id=12345

Wrapper xử lý:
1. Nhận request REST format
2. Trích xuất product_id = 12345
3. Gọi WMS API: GetInventory(SKU="12345")
4. Nhận response từ WMS (XML format)
5. Parse XML và convert sang JSON
6. Trả về cho e-commerce: {"product_id": 12345, "quantity": 150}
3. Cách Wrapper giải quyết vấn đề tương thích giao diện
a) Tách biệt phụ thuộc (Decoupling):

Hệ thống mới không cần biết chi tiết implementation của WMS cũ
WMS cũ không cần thay đổi để phục vụ hệ thống mới
Cho phép nâng cấp/thay thế một bên mà không ảnh hưởng bên kia

b) Chuyển đổi giao thức và dữ liệu:

Xử lý sự khác biệt về protocol (HTTP/REST vs SOAP)
Chuyển đổi định dạng dữ liệu (JSON ↔ XML)
Mapping các field names và data types khác nhau

c) Quản lý lỗi và retry logic:

Xử lý timeout của WMS cũ
Implement retry mechanism
Cung cấp error messages thống nhất cho e-commerce

4. So sánh chi phí phát triển: O(N²) Wrappers vs Message Broker
Mô hình A: Direct Wrappers (O(N²))
Nếu công ty có N hệ thống cần tích hợp với nhau:

Mỗi cặp hệ thống cần 1 wrapper riêng
Tổng số wrappers cần phát triển: N × (N-1) / 2 ≈ O(N²)

Ví dụ: Với 5 hệ thống (WMS, E-commerce, ERP, CRM, Accounting):

Số kết nối cần thiết: 5 × 4 / 2 = 10 wrappers
Với 10 hệ thống: 10 × 9 / 2 = 45 wrappers

Chi phí:

Phát triển: 10 wrappers × 20 giờ/wrapper = 200 giờ
Bảo trì: Mỗi thay đổi một hệ thống có thể ảnh hưởng 4 wrappers khác
Testing: Cần test 10 integration points
Độ phức tạp tăng theo cấp bậc hai khi thêm hệ thống mới

Mô hình B: Message Broker (O(N))
Sử dụng message broker làm hub trung tâm:
         ┌─── E-commerce
         │
WMS ────┤      Message Broker      ├──── ERP
         │    (RabbitMQ/Kafka)
         ├─── CRM
         │
         └─── Accounting
Mỗi hệ thống chỉ cần:

1 adapter kết nối tới message broker
Publish messages theo định dạng chuẩn
Subscribe các messages cần thiết

Chi phí:

Phát triển: 5 adapters × 15 giờ/adapter = 75 giờ
Setup message broker: 20 giờ
Tổng: 95 giờ (so với 200 giờ của mô hình A)
Thêm hệ thống mới: chỉ cần phát triển 1 adapter (~15 giờ)

Bảng so sánh chi tiết:
Tiêu chíO(N²) Direct WrappersMessage Broker O(N)Số components10 wrappers (5 hệ thống)5 adapters + 1 brokerChi phí phát triển ban đầuCao (200 giờ)Trung bình (95 giờ)Chi phí bảo trìRất caoThấpKhả năng mở rộngKém (thêm 1 hệ thống → 4 wrappers mới)Tốt (thêm 1 adapter)Single point of failureKhôngCó (broker)Độ phức tạpTăng theo N²Tăng tuyến tínhAsync processingKhó implementTự nhiên hỗ trợMessage queuingKhông cóCó sẵn
5. Khuyến nghị
Nên dùng Direct Wrapper khi:

Chỉ tích hợp 2-3 hệ thống
Cần giao tiếp synchronous (real-time)
Không có kế hoạch mở rộng nhiều hệ thống

Nên dùng Message Broker khi:

Có từ 4 hệ thống trở lên
Cần khả năng mở rộng trong tương lai
Chấp nhận được eventual consistency
Cần tính năng như message queuing, retry, dead-letter queue

Giải pháp hybrid (đề xuất cho bài toán):

Dùng wrapper đơn giản để connect E-commerce → Message Broker
Dùng adapter connect WMS → Message Broker
Khi cần tích hợp thêm ERP, CRM chỉ cần thêm adapter, không ảnh hưởng code hiện tại