# FlashSaleX - Task Tracker

Ký hiệu: `[x]` đã có trong code hoặc tài liệu; `[ ]` chưa hoàn thành.

## Trạng thái hiện tại

- Đã hoàn thành và merge PR #1: foundation migration-first cho `user-service`.
- Tiến độ hiện tại: `M0` hoàn thành `7/8` task.
- Task chưa hoàn thành gần nhất: thêm chiến lược integration test tự động với database thực.
- Sau khi hoàn thành test automation, feature nghiệp vụ tiếp theo là `M1 - User Service`, bắt đầu từ entity và repository.
- Không triển khai Redis, Gateway hoặc RabbitMQ trước khi hoàn thành luồng user và reservation core.

## Quy trình Git bắt buộc

- Không code trực tiếp trên `main`; mỗi nhóm công việc dùng một branch riêng.
- Trước khi commit: chạy test liên quan và kiểm tra `git status`.
- Mỗi commit chỉ chứa một mục đích rõ ràng: persistence, API/service, test hoặc docs/config.
- Sau khi feature pass test: push branch, mở Pull Request, review rồi mới merge vào `main`.
- Sau khi merge: cập nhật local `main`, xóa branch đã hoàn thành và tạo branch cho task tiếp theo.
- Chỉ đánh dấu `[x]` khi code/tài liệu đã commit và có bằng chứng kiểm tra tương ứng.

## M0 - Foundation

- [x] Thiết kế kiến trúc, database và API/event contract.
- [x] Chốt Java 21 + Spring Boot 4.0.6 làm baseline hiện tại.
- [x] Docker Compose khai báo PostgreSQL, Redis, RabbitMQ và healthcheck.
- [x] `user-service` dùng cấu hình DB qua environment override.
- [x] `user-service` dùng UTC, Flyway và Hibernate `ddl-auto=validate`.
- [x] Migration `users`, `roles`, `user_roles` và seed role.
- [x] Khởi động `postgres-user` và chạy test/migration thành công (27/05/2026).
- [ ] Thêm chiến lược integration test tự động với database thực.

### Checkpoint Git của M0

- [x] Commit cấu hình migration-first cho `user-service`.
- [x] Commit cấu hình hạ tầng local và healthcheck.
- [x] Commit tài liệu kiến trúc/roadmap/task tracker.
- [x] Chạy verification: Compose hợp lệ, PostgreSQL healthy, `user-service` test pass.
- [x] Tạo và merge PR #1 vào `main`.
- [ ] Thực hiện Testcontainers trong branch phù hợp trước khi dựa vào test tự động cho feature mới.

## M1 - User Service

### M1.1 - Persistence Model

- [ ] Tạo branch `feature/user-register`.
- [ ] Tạo enum `UserStatus`.
- [ ] Tạo entity `User`, `Role` và mapping `user_roles`.
- [ ] Tạo `UserRepository`, `RoleRepository`.
- [ ] Kiểm tra mapping khớp schema Flyway bằng test/application startup.
- [ ] Commit khi hoàn thành persistence model: `feat: add user and role persistence model`.

### M1.2 - Register API

- [ ] Tạo DTO request/response cho register.
- [ ] Validate input, normalize lowercase email.
- [ ] Cấu hình BCrypt và lưu `password_hash`.
- [ ] Gán role mặc định `USER`.
- [ ] Implement `POST /api/v1/auth/register`.
- [ ] Commit khi endpoint register chạy được: `feat: implement user registration`.

### M1.3 - Error Handling Và Register Test

- [ ] Tạo global API error format.
- [ ] Xử lý validation error và `EMAIL_ALREADY_EXISTS`.
- [ ] Viết test register thành công, email trùng và password được hash.
- [ ] Chuyển test sang database tự động/Testcontainers nếu chưa thực hiện.
- [ ] Chạy `.\mvnw.cmd test` pass.
- [ ] Commit phần test/error: `test: cover user registration flow`.
- [ ] Push branch, mở PR register, review và merge vào `main`.

### M1.4 - Login Và JWT

- [ ] Tạo branch mới sau khi PR register đã merge, ví dụ `feature/user-login-jwt`.
- [ ] Login API và JWT key từ environment.
- [ ] JWT authentication/authorization cho `/users/me`.
- [ ] Global API error format bổ sung cho auth failure nếu cần.
- [ ] Integration tests cho login/token/`/users/me` và failure cases.
- [ ] Chạy test pass, commit theo phạm vi, mở PR và merge.

## M2 - Flash Sale Catalog

- [ ] Khởi tạo `flash-sale-service`.
- [ ] Migration cho product/campaign/item.
- [ ] Admin product/campaign/item API.
- [ ] Public active campaign/item API.
- [ ] Authorization và integration tests.

### Checkpoint Git của M2

- [ ] Branch: `feature/flash-sale-catalog`.
- [ ] Commit riêng cho skeleton/config/migration.
- [ ] Commit riêng cho API catalog và validation.
- [ ] Test pass, mở PR và chỉ merge khi API public/admin có bằng chứng kiểm tra.

## M3 - Reservation Correctness

- [ ] Migration reservation/outbox.
- [ ] Atomic decrement trong transaction.
- [ ] Idempotency key và unique business constraints.
- [ ] Reservation API.
- [ ] Concurrent integration test chứng minh không oversell.

### Checkpoint Git của M3

- [ ] Branch: `feature/reservation-atomic-stock`.
- [ ] Commit schema/constraint trước khi viết transaction nghiệp vụ.
- [ ] Commit atomic reservation API sau khi test case cơ bản pass.
- [ ] Commit concurrent integration test riêng.
- [ ] PR phải ghi số stock ban đầu, số request đồng thời và kết quả accepted/remaining thực tế.

## M4 - Async Order

- [ ] Khởi tạo `order-service`.
- [ ] Schema order/processed event.
- [ ] RabbitMQ topology và outbox publisher.
- [ ] Idempotent order consumer.
- [ ] Order query API.
- [ ] Redelivery/retry/DLQ tests.

### Checkpoint Git của M4

- [ ] Branch riêng cho outbox publisher và branch riêng cho order consumer nếu diff lớn.
- [ ] Không merge publisher nếu chưa có retry/failure behavior tối thiểu được test.
- [ ] Không merge consumer nếu chưa test event giao lặp không tạo order trùng.
- [ ] PR ghi rõ delivery semantics và constraint idempotency sử dụng.

## M5 - Gateway và Vận Hành

- [ ] Khởi tạo `gateway-service`.
- [ ] Routing, CORS và correlation ID.
- [ ] Security model: downstream tự xác minh JWT trong MVP.
- [ ] Dockerfile, health endpoints và end-to-end test.

### Checkpoint Git của M5

- [ ] Branch: `feature/gateway-routing` hoặc tách vận hành thành PR riêng.
- [ ] Review security boundary trước khi merge; không tin identity header từ client.
- [ ] Có end-to-end verification qua gateway trước khi đánh dấu hoàn thành.

## M6 - Performance Evidence

- [ ] Script k6 baseline.
- [ ] Redis cache/read optimization.
- [ ] Rate limiting.
- [ ] Tối ưu stock path chỉ sau baseline và recovery design.
- [ ] Report có thể tái tạo để dùng làm bằng chứng CV.

### Checkpoint Git của M6

- [ ] Branch: `perf/k6-baseline` cho số đo gốc trước tối ưu.
- [ ] Commit script và report baseline trước khi thêm Redis.
- [ ] Tách Redis optimization thành PR riêng có báo cáo so sánh trước/sau.
- [ ] Chỉ đưa con số vào README/CV sau khi report đã merge vào `main`.
