# FlashSaleX - Task Tracker

Ký hiệu: `[x]` đã có trong code hoặc tài liệu; `[ ]` chưa hoàn thành.

## M0 - Foundation

- [x] Thiết kế kiến trúc, database và API/event contract.
- [x] Chốt Java 21 + Spring Boot 4.0.6 làm baseline hiện tại.
- [x] Docker Compose khai báo PostgreSQL, Redis, RabbitMQ và healthcheck.
- [x] `user-service` dùng cấu hình DB qua environment override.
- [x] `user-service` dùng UTC, Flyway và Hibernate `ddl-auto=validate`.
- [x] Migration `users`, `roles`, `user_roles` và seed role.
- [x] Khởi động `postgres-user` và chạy test/migration thành công (27/05/2026).
- [ ] Thêm chiến lược integration test tự động với database thực.

## M1 - User Service

- [ ] Entity và repository cho user/role.
- [ ] Register API: validation, lowercase email, BCrypt, role `USER`.
- [ ] Login API và JWT key từ environment.
- [ ] JWT authentication/authorization cho `/users/me`.
- [ ] Global API error format.
- [ ] Integration tests cho auth flow và failure cases.

## M2 - Flash Sale Catalog

- [ ] Khởi tạo `flash-sale-service`.
- [ ] Migration cho product/campaign/item.
- [ ] Admin product/campaign/item API.
- [ ] Public active campaign/item API.
- [ ] Authorization và integration tests.

## M3 - Reservation Correctness

- [ ] Migration reservation/outbox.
- [ ] Atomic decrement trong transaction.
- [ ] Idempotency key và unique business constraints.
- [ ] Reservation API.
- [ ] Concurrent integration test chứng minh không oversell.

## M4 - Async Order

- [ ] Khởi tạo `order-service`.
- [ ] Schema order/processed event.
- [ ] RabbitMQ topology và outbox publisher.
- [ ] Idempotent order consumer.
- [ ] Order query API.
- [ ] Redelivery/retry/DLQ tests.

## M5 - Gateway và Vận Hành

- [ ] Khởi tạo `gateway-service`.
- [ ] Routing, CORS và correlation ID.
- [ ] Security model: downstream tự xác minh JWT trong MVP.
- [ ] Dockerfile, health endpoints và end-to-end test.

## M6 - Performance Evidence

- [ ] Script k6 baseline.
- [ ] Redis cache/read optimization.
- [ ] Rate limiting.
- [ ] Tối ưu stock path chỉ sau baseline và recovery design.
- [ ] Report có thể tái tạo để dùng làm bằng chứng CV.
