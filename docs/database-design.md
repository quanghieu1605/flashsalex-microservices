# Thiết kế Database

## 1. Nguyên tắc

- Mỗi service sở hữu một PostgreSQL database riêng.
- ID dùng UUID để service có thể sinh ID mà không phụ thuộc sequence của
  database khác.
- Thời gian lưu dưới dạng UTC (`TIMESTAMP WITH TIME ZONE`); chuyển múi giờ ở
  lớp hiển thị.
- Schema được quản lý bằng Flyway migration, không dùng Hibernate tự sửa
  schema ở môi trường ngoài phát triển cá nhân.
- Không có foreign key giữa database của các service. Khi cần tham chiếu, lưu
  ID và snapshot dữ liệu cần thiết.

## 2. Database ownership

| Database | Owner | Dữ liệu chính |
| --- | --- | --- |
| `user_db` | `user-service` | Users, roles |
| `flashsale_db` | `flash-sale-service` | Products, campaigns, sale items, reservations, outbox |
| `order_db` | `order-service` | Orders, processed events |

## 3. `user_db`

### Bảng `users`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Định danh user |
| `email` | `VARCHAR(255)` | NOT NULL, UNIQUE | Email đăng nhập đã normalize lowercase |
| `password_hash` | `VARCHAR(255)` | NOT NULL | BCrypt hash, không phải password thô |
| `full_name` | `VARCHAR(120)` | NOT NULL | Tên hiển thị |
| `status` | `VARCHAR(20)` | NOT NULL | `ACTIVE`, `DISABLED` |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Thời điểm tạo |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | Thời điểm sửa cuối |

### Bảng `roles`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `BIGSERIAL` | PK | ID nội bộ |
| `name` | `VARCHAR(30)` | NOT NULL, UNIQUE | `USER`, `ADMIN` |

### Bảng `user_roles`

| Cột | Kiểu | Ràng buộc |
| --- | --- | --- |
| `user_id` | `UUID` | PK phần 1, FK -> `users.id` |
| `role_id` | `BIGINT` | PK phần 2, FK -> `roles.id` |

### Index và invariant

- Unique index trên `lower(email)` để `User@example.com` và
  `user@example.com` không trở thành hai tài khoản.
- User mới mặc định có role `USER`; role `ADMIN` được seed hoặc gán có kiểm
  soát, không cho client tự đăng ký làm admin.

## 4. `flashsale_db`

### Bảng `products`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Product ID |
| `sku` | `VARCHAR(64)` | NOT NULL, UNIQUE | Mã sản phẩm |
| `name` | `VARCHAR(200)` | NOT NULL | Tên sản phẩm |
| `description` | `TEXT` | NULL | Nội dung mô tả |
| `regular_price` | `NUMERIC(19,2)` | NOT NULL, CHECK >= 0 | Giá thường |
| `active` | `BOOLEAN` | NOT NULL | Có thể đưa vào campaign hay không |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Ngày tạo |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | Ngày cập nhật |

### Bảng `flash_sale_campaigns`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Campaign ID |
| `name` | `VARCHAR(150)` | NOT NULL | Tên đợt sale |
| `start_at` | `TIMESTAMPTZ` | NOT NULL | Giờ bắt đầu |
| `end_at` | `TIMESTAMPTZ` | NOT NULL, CHECK `end_at > start_at` | Giờ kết thúc |
| `status` | `VARCHAR(20)` | NOT NULL | `DRAFT`, `SCHEDULED`, `CANCELLED` |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Ngày tạo |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | Ngày cập nhật |

Trạng thái `OPEN` và `ENDED` có thể suy ra từ thời gian khi campaign không bị
cancel. Cách này tránh job cập nhật status bị trễ làm sale đóng/mở sai giờ.

### Bảng `flash_sale_items`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Item được mua trong API |
| `campaign_id` | `UUID` | NOT NULL, FK nội bộ | Campaign |
| `product_id` | `UUID` | NOT NULL, FK nội bộ | Product |
| `sale_price` | `NUMERIC(19,2)` | NOT NULL, CHECK >= 0 | Giá bán trong sale |
| `initial_stock` | `INTEGER` | NOT NULL, CHECK >= 0 | Số lượng đưa vào sale |
| `remaining_stock` | `INTEGER` | NOT NULL, CHECK >= 0 | Số lượng còn bán được |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Ngày tạo |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | Ngày cập nhật |

Ràng buộc bổ sung:

- `UNIQUE(campaign_id, product_id)`: một sản phẩm xuất hiện một lần trong một
  campaign.
- `CHECK(sale_price <= regular_price)` không đặt trực tiếp được nếu giá thường
  nằm ở bảng khác; service kiểm tra khi tạo item và order lưu snapshot giá.
- Không cho sửa `initial_stock` sau khi sale bắt đầu trong MVP.

### Bảng `reservations`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Reservation ID |
| `flash_sale_item_id` | `UUID` | NOT NULL, FK nội bộ | Item được giữ chỗ |
| `user_id` | `UUID` | NOT NULL | ID từ JWT; không FK sang `user_db` |
| `idempotency_key` | `VARCHAR(100)` | NOT NULL | Key retry của client |
| `quantity` | `INTEGER` | NOT NULL, CHECK = 1 | MVP chỉ mua một đơn vị |
| `unit_price` | `NUMERIC(19,2)` | NOT NULL | Snapshot giá lúc mua |
| `status` | `VARCHAR(30)` | NOT NULL | `ACCEPTED`, `CANCELLED`, `EXPIRED` |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Lúc giữ chỗ |

Index và invariant:

- `UNIQUE(user_id, idempotency_key)` cho cùng request không được thực hiện lại.
- Partial unique index trên `(flash_sale_item_id, user_id)` khi status là
  `ACCEPTED` để một user không mua trùng item.
- Stock chỉ giảm khi insert reservation `ACCEPTED` thành công trong cùng
  transaction.

### Bảng `outbox_events`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Event ID |
| `aggregate_type` | `VARCHAR(50)` | NOT NULL | `RESERVATION` |
| `aggregate_id` | `UUID` | NOT NULL | Reservation ID |
| `event_type` | `VARCHAR(80)` | NOT NULL | `ReservationAccepted` |
| `payload` | `JSONB` | NOT NULL | Nội dung gửi RabbitMQ |
| `status` | `VARCHAR(20)` | NOT NULL | `PENDING`, `PUBLISHED`, `FAILED` |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Lúc event được ghi |
| `published_at` | `TIMESTAMPTZ` | NULL | Lúc publish thành công |
| `retry_count` | `INTEGER` | NOT NULL DEFAULT 0 | Số lần thử gửi |

Index cần có: `(status, created_at)` để publisher lấy nhanh event `PENDING`.

## 5. `order_db`

### Bảng `orders`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `id` | `UUID` | PK | Order ID |
| `reservation_id` | `UUID` | NOT NULL, UNIQUE | Bảo vệ không tạo order lặp |
| `user_id` | `UUID` | NOT NULL | Chủ đơn hàng |
| `flash_sale_item_id` | `UUID` | NOT NULL | Item nguồn |
| `quantity` | `INTEGER` | NOT NULL | Số lượng đã giữ chỗ |
| `unit_price` | `NUMERIC(19,2)` | NOT NULL | Snapshot từ event |
| `total_amount` | `NUMERIC(19,2)` | NOT NULL | `quantity * unit_price` |
| `status` | `VARCHAR(30)` | NOT NULL | `CREATED`, `CANCELLED`, `PAID` |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | Ngày tạo |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | Ngày cập nhật |

### Bảng `processed_events`

| Cột | Kiểu | Ràng buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `event_id` | `UUID` | PK | Event đã xử lý |
| `event_type` | `VARCHAR(80)` | NOT NULL | Loại event |
| `processed_at` | `TIMESTAMPTZ` | NOT NULL | Thời điểm xử lý |

`orders.reservation_id` là bảo vệ nghiệp vụ; `processed_events.event_id` là
bảo vệ consumer khi RabbitMQ redeliver cùng một event.

## 6. Transaction quan trọng nhất

Pseudo SQL cho reservation:

```sql
begin;

update flash_sale_items
set remaining_stock = remaining_stock - 1,
    updated_at = now()
where id = :item_id
  and remaining_stock >= 1;

-- Nếu affected rows = 0: rollback và trả SOLD_OUT.

insert into reservations (..., status)
values (..., 'ACCEPTED');

insert into outbox_events (..., event_type, status)
values (..., 'ReservationAccepted', 'PENDING');

commit;
```

Nếu unique constraint của reservation bị vi phạm, toàn transaction rollback,
bao gồm phép trừ stock. Đây là lý do stock và reservation phải nằm cùng
database của `flash-sale-service`.

## 7. Migration plan

| Giai đoạn | Service | Migration |
| --- | --- | --- |
| Đã tạo | user | `V1__create_users_and_roles.sql`, `V2__seed_roles.sql` |
| Dự kiến | flash sale | `V1__create_products_campaigns_and_items.sql` |
| Dự kiến | flash sale | `V2__create_reservations_and_outbox.sql` |
| Dự kiến | order | `V1__create_orders_and_processed_events.sql` |

`user-service` đã cấu hình `spring.jpa.hibernate.ddl-auto=validate`. Các
service tiếp theo phải theo cùng quy ước: Flyway thay đổi schema, ứng dụng chỉ
kiểm tra mapping phù hợp schema.
