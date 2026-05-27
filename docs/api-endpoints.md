# API và Event Contract

## 1. Quy ước chung

### Base path

Tất cả HTTP endpoint sử dụng prefix `/api/v1`.

### Authentication

- Endpoint public: đăng ký, đăng nhập, xem campaign/item đang bán.
- Endpoint authenticated: tạo reservation, xem order của chính user.
- Endpoint admin: tạo/sửa product, campaign, item.
- Trong MVP, service sở hữu endpoint tự xác minh chữ ký và claims JWT.
- Gateway không được khiến downstream tin `X-User-Id`/`X-User-Roles` từ
  request client nếu chưa có trust boundary nội bộ được bảo vệ.
- Request authenticated gửi header:

```http
Authorization: Bearer <access-token>
```

### Format response thành công

```json
{
  "data": {},
  "timestamp": "2026-05-24T10:00:00Z"
}
```

### Format lỗi

```json
{
  "code": "SOLD_OUT",
  "message": "Flash sale item is sold out",
  "details": [],
  "timestamp": "2026-05-24T10:00:00Z",
  "path": "/api/v1/flash-sale-items/0c36.../reservations"
}
```

| HTTP status | Khi dùng |
| --- | --- |
| `200 OK` | Query hoặc login thành công |
| `201 Created` | Tạo tài nguyên đồng bộ thành công |
| `202 Accepted` | Reservation được chấp nhận, order sẽ tạo bất đồng bộ |
| `400 Bad Request` | Input sai định dạng/validation |
| `401 Unauthorized` | Thiếu hoặc sai token |
| `403 Forbidden` | Đã đăng nhập nhưng thiếu role |
| `404 Not Found` | Resource không tồn tại |
| `409 Conflict` | Email trùng, mua trùng, idempotency conflict |
| `422 Unprocessable Entity` | Campaign chưa mở/đã kết thúc/hết hàng |

## 2. `user-service`

Service chạy local dự kiến tại `http://localhost:8081`.

### `POST /api/v1/auth/register`

Public. Tạo user có role mặc định `USER`.

Request:

```json
{
  "email": "nguyen@example.com",
  "password": "StrongPassword123!",
  "fullName": "Nguyen Van A"
}
```

Validation:

- `email` hợp lệ và tối đa 255 ký tự; lưu lowercase.
- `password` tối thiểu 8 ký tự; password hash bằng BCrypt trước khi lưu.
- `fullName` không rỗng, tối đa 120 ký tự.

Response `201 Created`:

```json
{
  "data": {
    "id": "6cd7d89e-321d-45c2-8141-8ca71b18cc1c",
    "email": "nguyen@example.com",
    "fullName": "Nguyen Van A",
    "roles": ["USER"]
  },
  "timestamp": "2026-05-24T10:00:00Z"
}
```

Errors: `EMAIL_ALREADY_EXISTS` (`409`), `VALIDATION_ERROR` (`400`).

### `POST /api/v1/auth/login`

Public. Xác minh credential và phát JWT.

Request:

```json
{
  "email": "nguyen@example.com",
  "password": "StrongPassword123!"
}
```

Response `200 OK`:

```json
{
  "data": {
    "accessToken": "<jwt>",
    "tokenType": "Bearer",
    "expiresIn": 3600
  },
  "timestamp": "2026-05-24T10:00:00Z"
}
```

JWT claim tối thiểu:

```json
{
  "sub": "6cd7d89e-321d-45c2-8141-8ca71b18cc1c",
  "roles": ["USER"],
  "iat": 1779616800,
  "exp": 1779620400
}
```

Errors: `INVALID_CREDENTIALS` (`401`), `USER_DISABLED` (`403`).

### `GET /api/v1/users/me`

Authenticated. Trả thông tin user lấy theo `sub` trong token, không nhận
`userId` từ query parameter.

## 3. `flash-sale-service`

Port local dự kiến: `8082`.

### Admin endpoints

| Method | Endpoint | Mục đích | Success |
| --- | --- | --- | --- |
| `POST` | `/api/v1/admin/products` | Tạo sản phẩm | `201` |
| `PUT` | `/api/v1/admin/products/{productId}` | Sửa sản phẩm | `200` |
| `POST` | `/api/v1/admin/campaigns` | Tạo campaign | `201` |
| `POST` | `/api/v1/admin/campaigns/{campaignId}/items` | Thêm item sale | `201` |
| `PUT` | `/api/v1/admin/flash-sale-items/{itemId}` | Sửa item trước khi mở bán | `200` |

Tất cả yêu cầu role `ADMIN`.

Ví dụ tạo campaign:

```json
{
  "name": "Laptop Opening Sale",
  "startAt": "2026-06-01T02:00:00Z",
  "endAt": "2026-06-01T03:00:00Z"
}
```

Ví dụ thêm item:

```json
{
  "productId": "43bfbff4-512f-4d98-ab04-97fe48865ed0",
  "salePrice": 9990000.00,
  "initialStock": 100
}
```

### `GET /api/v1/campaigns/active`

Public. Lấy campaign đang diễn ra theo thời gian hiện tại.

### `GET /api/v1/campaigns/{campaignId}/items`

Public. Hiển thị item, giá sale và trạng thái còn/hết hàng. API hiển thị stock
có thể chậm hơn thời điểm mua; kết quả cuối cùng chỉ được quyết định khi gọi
reservation endpoint.

### `POST /api/v1/flash-sale-items/{itemId}/reservations`

Authenticated. Đây là endpoint quan trọng nhất của hệ thống.

Headers:

```http
Authorization: Bearer <access-token>
Idempotency-Key: 79aa54b4-2d06-4b60-986c-24e548354e8d
```

Request body:

```json
{
  "quantity": 1
}
```

Quy tắc:

- `userId` lấy từ JWT.
- `quantity` trong MVP luôn phải bằng `1`.
- Campaign phải đang mở.
- Một user chỉ được reservation thành công một lần trên một item.
- Cùng `Idempotency-Key` được retry và trả cùng kết quả.

Response `202 Accepted`:

```json
{
  "data": {
    "reservationId": "34ba1204-f9be-4765-93b3-577423acf559",
    "status": "ACCEPTED",
    "flashSaleItemId": "acb20974-d780-4c7d-b9af-cb6aa8173481",
    "quantity": 1,
    "unitPrice": 9990000.00
  },
  "timestamp": "2026-06-01T02:00:01Z"
}
```

Business errors:

| Code | HTTP | Ý nghĩa |
| --- | --- | --- |
| `SALE_NOT_STARTED` | `422` | Chưa tới giờ sale |
| `SALE_ENDED` | `422` | Sale đã kết thúc |
| `SOLD_OUT` | `422` | Không còn stock để giữ chỗ |
| `ALREADY_RESERVED` | `409` | User đã mua item này |
| `IDEMPOTENCY_KEY_REUSED` | `409` | Key cũ được dùng với request khác |

### `GET /api/v1/reservations/{reservationId}`

Authenticated. Chỉ user sở hữu reservation hoặc admin được xem. Dùng cho
client kiểm tra trạng thái trong lúc order đang được tạo bất đồng bộ.

## 4. `order-service`

Port local dự kiến: `8083`.

### `GET /api/v1/orders`

Authenticated. Liệt kê order của user lấy từ JWT.

### `GET /api/v1/orders/{orderId}`

Authenticated. Trả order của chính user hoặc cho admin.

Order không được tạo trực tiếp bằng public HTTP API trong MVP. Nó được tạo từ
event `ReservationAccepted`, nhờ vậy chỉ reservation đã chiếm stock mới trở
thành order.

## 5. `gateway-service`

Port local dự kiến: `8080`.

Trong MVP gateway route request và có thể kiểm tra token sớm, nhưng vẫn forward
header `Authorization` để service đích tự authorize. Phương án gateway tạo
trusted identity header chỉ được cân nhắc khi service không thể bị gọi vòng
qua gateway và traffic nội bộ đã được xác thực.

| External path | Routed service |
| --- | --- |
| `/api/v1/auth/**`, `/api/v1/users/**` | `user-service` |
| `/api/v1/campaigns/**`, `/api/v1/flash-sale-items/**`, `/api/v1/reservations/**` | `flash-sale-service` |
| `/api/v1/orders/**` | `order-service` |
| `/api/v1/admin/products/**`, `/api/v1/admin/campaigns/**`, `/api/v1/admin/flash-sale-items/**` | `flash-sale-service` |

Swagger/OpenAPI của mỗi service có thể giữ riêng trong giai đoạn đầu. Sau khi
gateway hoạt động mới cân nhắc gom documentation.

## 6. RabbitMQ event contract

### Exchange và routing

| Thành phần | Giá trị dự kiến |
| --- | --- |
| Exchange | `flashsalex.events` |
| Exchange type | `topic` |
| Routing key | `reservation.accepted.v1` |
| Order queue | `order.reservation-accepted.v1` |
| Dead letter queue | `order.reservation-accepted.dlq.v1` |

### Event `ReservationAccepted.v1`

Publisher: `flash-sale-service`

Consumer: `order-service`

```json
{
  "eventId": "dfe753bd-abdb-49c0-a180-1c8179e2d54d",
  "eventType": "ReservationAccepted",
  "eventVersion": 1,
  "occurredAt": "2026-06-01T02:00:01Z",
  "data": {
    "reservationId": "34ba1204-f9be-4765-93b3-577423acf559",
    "userId": "6cd7d89e-321d-45c2-8141-8ca71b18cc1c",
    "flashSaleItemId": "acb20974-d780-4c7d-b9af-cb6aa8173481",
    "productId": "43bfbff4-512f-4d98-ab04-97fe48865ed0",
    "productName": "Laptop Pro 14",
    "quantity": 1,
    "unitPrice": 9990000.00
  }
}
```

Event chứa snapshot `productName` và `unitPrice` vì order cần lưu lịch sử tại
thời điểm mua, không gọi ngược sang database/service khác để dựng lại giá cũ.

### Consumer rules

- Consumer lưu `eventId` vào `processed_events`.
- `reservationId` có unique constraint trong bảng `orders`.
- Nếu event được deliver hai lần, lần thứ hai được acknowledge mà không tạo
  order mới.
- Lỗi tạm thời được retry; lỗi vượt ngưỡng chuyển DLQ để điều tra.

## 7. Thứ tự triển khai contract

| Tuần | API/event được hiện thực |
| --- | --- |
| 2-3 | Register, login, `/users/me` |
| 4 | Product/campaign/item admin và query public |
| 5 | Reservation endpoint |
| 7 | `ReservationAccepted.v1`, order query |
| 8 | Routing qua gateway |
