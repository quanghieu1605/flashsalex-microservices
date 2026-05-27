# Kế Hoạch Triển Khai FlashSaleX

Tài liệu này là roadmap triển khai, không phải danh sách thành tựu đã hoàn
thành. Mục tiêu của dự án là xây một backend có thể giải thích và chứng minh
được các quyết định về concurrency, consistency, security và vận hành.

## 1. Trạng Thái Hiện Tại

Đã có:

- Java 21, Spring Boot 4.0.6 và skeleton `user-service`.
- Docker Compose cho PostgreSQL, Redis, RabbitMQ.
- Flyway migration đầu tiên của `user-service`.
- Thiết kế kiến trúc, database và HTTP/event contract.

Chưa có:

- API register/login/JWT.
- Flash sale, reservation, outbox hoặc order processing.
- Gateway, Redis integration, k6 report.
- Kết quả có thể dùng làm achievement trong CV.

## 2. Nguyên Tắc Kỹ Thuật

1. Tính đúng trước tối ưu: PostgreSQL bảo vệ stock trước khi đưa Redis vào hot path.
2. Database-per-service: không query/FK xuyên database.
3. Migration-first: Flyway thay đổi schema, Hibernate dùng `validate`.
4. Security theo defense in depth: business service tự authorize JWT ở MVP.
5. Có bằng chứng trước khi tuyên bố: mỗi milestone cần test hoặc report tái tạo được.

## 3. Milestone Triển Khai

### M0 - Foundation đang thực hiện

Mục tiêu: source code, cấu hình và tài liệu không tự mâu thuẫn.

Task:

- [x] Chốt baseline Java 21 + Spring Boot 4.0.6.
- [x] Đưa `user-service` về UTC, environment-based config và Flyway.
- [x] Tạo migration user/role đầu tiên.
- [x] Bổ sung healthcheck hạ tầng local.
- [x] Gỡ các tuyên bố CV chưa được chứng minh khỏi roadmap.
- [x] Chạy Compose và xác minh migration/test hoạt động trên PostgreSQL (27/05/2026).
- [ ] Chọn chiến lược integration test tự động, ưu tiên Testcontainers.

Bài học cần nắm: vì sao `ddl-auto=update` không phù hợp production và vì sao
UTC là quy ước cần giữ giữa các service.

### M1 - User Service và Authentication

Mục tiêu: người dùng đăng ký, đăng nhập và gọi endpoint được bảo vệ.

Task:

- [ ] Tạo entity `User`, `Role` và repository.
- [ ] Tạo DTO validation, normalize email và hash password bằng BCrypt.
- [ ] Implement `POST /api/v1/auth/register`.
- [ ] Implement `POST /api/v1/auth/login` và JWT access token.
- [ ] Implement security filter/config và `GET /api/v1/users/me`.
- [ ] Implement error response chung theo API contract.
- [ ] Viết integration test cho email trùng, credential sai và token hợp lệ.

Bằng chứng hoàn thành: test chạy tự động cho register -> login -> `/users/me`;
không lưu/log password thô; key JWT lấy từ environment.

### M2 - Flash Sale Catalog

Mục tiêu: admin chuẩn bị campaign và user đọc danh sách sale.

Task:

- [ ] Khởi tạo `flash-sale-service` theo cùng baseline/config convention.
- [ ] Flyway migration cho `products`, `flash_sale_campaigns`, `flash_sale_items`.
- [ ] Admin authorization cho API tạo/sửa dữ liệu.
- [ ] Public query cho campaign active và item.
- [ ] Test validation: thời gian campaign, giá sale và stock.

Bằng chứng hoàn thành: API test tạo campaign và query item qua PostgreSQL riêng.

### M3 - Reservation và Chống Overselling

Mục tiêu: giải quyết bài toán quan trọng nhất của dự án.

Task:

- [ ] Migration cho `reservations` và `outbox_events`.
- [ ] Conditional atomic update giảm `remaining_stock`.
- [ ] Transaction ghi stock, reservation và outbox cùng lúc.
- [ ] Unique constraint một user/một item và idempotency key.
- [ ] `POST /api/v1/flash-sale-items/{itemId}/reservations`.
- [ ] Concurrent integration test với stock hữu hạn.

Bằng chứng hoàn thành: dưới nhiều request đồng thời, số reservation accepted
không vượt stock và `remaining_stock` không âm.

### M4 - Event và Order Service

Mục tiêu: tạo order bất đồng bộ mà không mất hoặc nhân đôi nghiệp vụ.

Task:

- [ ] Khởi tạo `order-service` và migration `orders`, `processed_events`.
- [ ] Cấu hình exchange, queue, retry/DLQ.
- [ ] Outbox publisher có publisher confirm và retry.
- [ ] Consumer idempotent bằng `event_id` và `reservation_id`.
- [ ] API xem order của user.
- [ ] Test redelivery và broker tạm lỗi.

Bằng chứng hoàn thành: event giao lặp không tạo order lặp; outbox chưa publish
thành công vẫn còn khả năng retry.

### M5 - Gateway và Observability

Mục tiêu: một entry point và luồng truy vết rõ ràng.

Task:

- [ ] Khởi tạo `gateway-service` và routing.
- [ ] CORS, request/correlation ID và log structure.
- [ ] Gateway có thể reject token sai; downstream vẫn tự authorize JWT.
- [ ] Health endpoint và Dockerfile cho từng service.

Bằng chứng hoàn thành: end-to-end request qua gateway và log truy được
`requestId`, `reservationId`, `eventId`, `orderId`.

### M6 - Redis và Đo Tải

Mục tiêu: tối ưu dựa trên baseline, không đánh đổi tính đúng.

Task:

- [ ] Viết k6 baseline cho reservation dùng PostgreSQL.
- [ ] Cache read path và đo lại.
- [ ] Rate limit request mua.
- [ ] Chỉ thử fast sold-out gate khi có thiết kế recovery rõ ràng.
- [ ] Lưu report throughput, latency, error rate và correctness assertions.

Bằng chứng hoàn thành: report có command/config chạy lại, không dùng các con
số ước lượng trong CV.

## 4. Cách Viết CV Sau Khi Có Bằng Chứng

Hiện tại có thể mô tả dự án là đang xây dựng một flash-sale backend tập trung
vào concurrency và event-driven consistency. Không ghi đã ngăn overselling,
đã tối ưu phần trăm tải hoặc đã chịu 1.000 concurrent users cho đến khi các
milestone và report tương ứng hoàn thành.

Khi đã hoàn thành, mỗi bullet CV cần có:

- Bài toán: ví dụ race condition khi reserve stock.
- Quyết định: atomic SQL update, outbox hoặc idempotent consumer.
- Bằng chứng: test/load report và con số thực đo được.
