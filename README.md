# FlashSaleX Microservices

FlashSaleX là dự án backend mô phỏng một đợt flash sale có lượng yêu cầu mua
đồng thời cao. Mục tiêu kỹ thuật là chứng minh hệ thống chỉ chấp nhận số lượt
giữ chỗ không vượt quá tồn kho, sau đó tạo đơn hàng bất đồng bộ và có thể truy
vết khi xảy ra retry hoặc lỗi hạ tầng.

## Stack đã chọn

- Java 21 và Spring Boot 4.0.6.
- PostgreSQL 16 làm nguồn sự thật cho dữ liệu và tồn kho.
- Flyway quản lý migration; Hibernate chỉ kiểm tra mapping/schema.
- RabbitMQ cho event tạo order bất đồng bộ.
- Redis chỉ dành cho cache/rate limiting hoặc tối ưu được đo sau baseline.
- Docker Compose cho hạ tầng local; k6 dự kiến dùng để đo tải.

## Trạng thái triển khai

Đã hiện thực trong repository:

- Tài liệu kiến trúc, database và API/event contract trong `docs/`.
- Docker Compose cho ba PostgreSQL database, Redis và RabbitMQ, kèm healthcheck.
- Skeleton `user-service` chạy tại port `8081`.
- Flyway migration đầu tiên cho bảng `users`, `roles`, `user_roles` và seed role.
- Cấu hình `user-service` dùng UTC, environment override và
  `spring.jpa.hibernate.ddl-auto=validate`.

Chưa hiện thực:

- Register, login, JWT, authorization và API người dùng.
- `flash-sale-service`, reservation/atomic stock decrement và outbox publisher.
- `order-service`, RabbitMQ consumer idempotent.
- `gateway-service`, Redis optimization, automated integration/load test.
- Bất kỳ số liệu latency, throughput hoặc kết quả chống overselling nào.

Các kết quả hiệu năng hoặc bullet CV chỉ được viết như thành tựu sau khi có
source code, test chạy lại được và report đo lường lưu trong repository.

## Thiết kế mục tiêu

```text
Client
  -> User Service       -> user_db
  -> Flash Sale Service -> flashsale_db -> outbox -> RabbitMQ
                                           |
                                           v
                                      Order Service -> order_db
```

Nguyên tắc chính:

- Mỗi service sở hữu database riêng.
- PostgreSQL bảo vệ invariant tồn kho bằng conditional atomic update.
- Reservation và outbox event được ghi trong cùng transaction.
- Order consumer xử lý event lặp lại một cách idempotent.
- Ở MVP, mỗi business service tự xác minh JWT cho endpoint được bảo vệ; không
  tin header định danh người dùng do client có thể tự gửi.
- Redis không là nguồn sự thật cho stock trong baseline.

## Tài liệu

- [Kiến trúc hệ thống](docs/architecture.md)
- [Thiết kế database](docs/database-design.md)
- [API và event contract](docs/api-endpoints.md)
- [Kế hoạch triển khai](implementation_plan.md)
- [Bảng task](task.md)

## Chạy local

Tạo file `.env` từ `.env.example` nếu muốn thay đổi credential local, sau đó:

```bash
docker compose up -d
cd user-service
./mvnw test
```

Trên Windows PowerShell dùng `.\mvnw.cmd test`.

Maven test được cấu hình chạy JVM theo UTC để kết nối PostgreSQL ổn định ngay
cả trên máy có timezone hệ điều hành dùng alias không được PostgreSQL nhận diện.

| Thành phần | Cổng host |
| --- | --- |
| PostgreSQL - user | `5433` |
| PostgreSQL - flash sale | `5434` |
| PostgreSQL - order | `5435` |
| Redis | `6379` |
| RabbitMQ AMQP | `5672` |
| RabbitMQ Management UI | `15672` |
