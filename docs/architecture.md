# Kiến trúc FlashSaleX

## 1. Bài toán

Trong một flash sale, số lượng sản phẩm giới hạn nhưng số request mua có thể
đến gần như cùng lúc. Nếu hai request cùng đọc `remaining_stock = 1` rồi đều
ghi thành công, hệ thống đã bán hai sản phẩm dù chỉ có một sản phẩm. Đây là
race condition dẫn đến **overselling**.

FlashSaleX tập trung xử lý câu hỏi:

> Làm thế nào nhận nhiều yêu cầu mua đồng thời, chỉ chấp nhận đúng số lượng
> được phép bán, rồi tạo đơn hàng một cách tin cậy?

### Phạm vi MVP

- Người dùng đăng ký và đăng nhập.
- Admin tạo sản phẩm và chiến dịch sale.
- Mỗi người dùng chỉ mua tối đa một đơn vị của một mặt hàng sale.
- Hệ thống giữ chỗ tồn kho trước, sau đó tạo order bất đồng bộ.
- Có test đồng thời chứng minh số reservation thành công không vượt stock.

### Chưa thuộc MVP

- Thanh toán thật, vận chuyển, hoàn tiền.
- Multi-region deployment hoặc Kubernetes.
- Giỏ hàng gồm nhiều mặt hàng trong một lần checkout.
- Dynamic pricing hoặc recommendation.

Giới hạn phạm vi giúp phần quan trọng nhất, concurrency và consistency, được
triển khai đúng và có thể giải thích rõ khi phỏng vấn.

## 2. Kiến trúc mức cao

```text
Client
  |
  v
Gateway Service  ---- validates/routes public requests
  |                  |
  |                  +---- User Service ------ PostgreSQL: user_db
  |
  +---- Flash Sale Service -------------------- PostgreSQL: flashsale_db
            |       |
            |       +-------------------------- Redis
            |
            +---- publishes ReservationAccepted through RabbitMQ
                                                     |
                                                     v
                                           Order Service ---- PostgreSQL: order_db
```

### Nguyên tắc ownership

Mỗi service là chủ sở hữu duy nhất của database của nó. Service khác không
query trực tiếp database đó và không tạo foreign key xuyên database. Giao tiếp
qua HTTP API hoặc event.

Lý do:

- Service có thể thay đổi schema nội bộ mà không phá service khác.
- Tránh coupling ngầm qua database chung.
- Luồng lỗi mạng/message được nhìn thấy và xử lý rõ ràng.

## 3. Trách nhiệm từng service

| Service | Trách nhiệm | Không chịu trách nhiệm |
| --- | --- | --- |
| `gateway-service` | Entry point, routing, CORS, có thể từ chối token sai sớm | Không là trust boundary duy nhất, không lưu user/order/stock |
| `user-service` | User account, password hash, role, login, phát JWT | Không quyết định tồn kho |
| `flash-sale-service` | Product, campaign, sale item, reservation, kiểm soát stock | Không quản lý vòng đời order |
| `order-service` | Tạo order từ reservation thành công, trạng thái order | Không tự trừ tồn kho sale |

Trong MVP, mỗi service nghiệp vụ tự xác minh JWT và lấy `userId`/role từ
claims đã xác minh. Gateway có thể từ chối token sai sớm, nhưng không biến các
header như `X-User-Id` do client gửi thành danh tính tin cậy. Cách chỉ tin
header nội bộ chỉ phù hợp khi đã có network isolation và cơ chế xác thực
service-to-service được triển khai, không phải trạng thái hiện tại.

## 4. Luồng nghiệp vụ

### 4.1 Đăng ký và đăng nhập

```text
Client -> User Service: POST /api/v1/auth/register
User Service -> user_db: insert user với password đã BCrypt hash
Client -> User Service: POST /api/v1/auth/login
User Service -> Client: access token JWT
```

JWT mang `sub` là `userId` và claim `roles`. Password thô không được lưu vào
database hoặc ghi log.

### 4.2 Admin chuẩn bị chiến dịch

```text
Admin -> Flash Sale Service: tạo product
Admin -> Flash Sale Service: tạo campaign có start/end time
Admin -> Flash Sale Service: thêm item với sale price và available stock
```

Một `flash_sale_item` gắn một sản phẩm vào một campaign và chứa tồn kho riêng
cho đợt bán đó. Stock sale không đồng nghĩa với tổng stock kho hàng của một
hệ thống thương mại điện tử hoàn chỉnh; đây là phạm vi được đơn giản hóa.

### 4.3 Khách hàng đặt mua

Luồng mục tiêu ở phiên bản bảo đảm đúng bằng PostgreSQL:

```text
Client -> Flash Sale Service: POST /api/v1/flash-sale-items/{itemId}/reservations
Flash Sale Service:
  1. Xác thực user và idempotency key.
  2. Kiểm tra campaign đang OPEN theo thời gian server.
  3. Trong một transaction, thực hiện atomic stock decrement:
       UPDATE flash_sale_items
       SET remaining_stock = remaining_stock - 1
       WHERE id = :id AND remaining_stock > 0
  4. Nếu số dòng update = 0: trả SOLD_OUT.
  5. Insert reservation ACCEPTED và outbox event.
  6. Trả 202 Accepted kèm reservationId.
Outbox publisher -> RabbitMQ: ReservationAccepted
Order Service -> order_db: tạo order duy nhất cho reservationId
```

Tại sao bước 3 quan trọng: database thực hiện điều kiện kiểm tra và phép trừ
trong cùng một statement nguyên tử. Hai request không thể cùng tiêu thụ đơn
vị cuối cùng. Nếu insert reservation hoặc outbox thất bại, transaction rollback
và stock được hoàn lại.

### 4.4 Idempotency

Client gửi header `Idempotency-Key` cho thao tác mua. Nếu client retry do
timeout, cùng một key phải nhận lại cùng kết quả thay vì mua thêm lần nữa.

Ngoài ra, MVP có constraint một user chỉ được có một reservation được chấp
nhận cho một `flashSaleItemId`. Consumer tạo order dùng `reservationId` làm
unique key để event được giao lại cũng không tạo hai order.

## 5. PostgreSQL, Redis và RabbitMQ dùng để làm gì

### PostgreSQL: nguồn sự thật

PostgreSQL lưu user, campaign, item stock, reservation và order. Invariant
`remaining_stock >= 0` phải được bảo vệ tại database và trong transaction.

### RabbitMQ: tách tạo order khỏi đường request nóng

Sau khi reservation thành công, client không cần đợi toàn bộ việc tạo order.
Message giúp `order-service` xử lý độc lập và scale consumer sau này.

Rủi ro: publish message trực tiếp sau khi commit database có thể mất event nếu
process chết giữa hai thao tác. Vì vậy target design dùng **transactional
outbox**: transaction ghi cả reservation và bản ghi event; publisher riêng đọc
outbox để gửi RabbitMQ, retry khi cần.

### Redis: tối ưu có chủ đích

Redis sẽ được thêm sau khi phiên bản đúng bằng database đã được đo:

- Cache campaign/item đang hiển thị nhiều.
- Rate limit request mua trên user/IP.
- Fast sold-out gate để giảm request vô ích khi hàng đã hết.

Ở MVP, Redis **không phải** nguồn sự thật cho stock. Nếu sau này thử reserve
stock bằng Redis Lua script để tăng throughput, cần thiết kế rõ cơ chế phục hồi
khi message/database thất bại. Đây là extension có thể đo so sánh, không phải
điều kiện để phiên bản đầu đúng.

## 6. Consistency và failure scenarios

| Trường hợp | Cách xử lý dự kiến |
| --- | --- |
| Hai user mua sản phẩm cuối cùng cùng lúc | Atomic SQL chỉ cho một request thành công |
| Cùng user bấm mua hai lần | Unique constraint và idempotency key ngăn mua trùng |
| Response bị timeout nhưng reservation đã ghi | Retry cùng key trả lại kết quả cũ |
| Reservation commit nhưng RabbitMQ tạm ngưng | Outbox publisher retry cho đến khi gửi được |
| RabbitMQ giao event lặp lại | Order consumer idempotent theo `reservationId` |
| Order tạo thất bại tạm thời | Consumer retry/DLQ; reservation vẫn truy vết được |

## 7. Security boundary

- Password được hash bằng BCrypt trong `user-service`.
- JWT access token được ký; secret/key lấy từ environment, không commit vào Git.
- Endpoint admin yêu cầu role `ADMIN`.
- Endpoint reservation yêu cầu user đăng nhập; `userId` lấy từ token, không tin
  giá trị do client gửi trong request body.
- Database password được chuyển sang environment variable trước khi triển khai.

## 8. Observability và kiểm thử

Mỗi request mua nên có `requestId`, `idempotencyKey` và `reservationId` để truy
vết log giữa service và message.

Các bằng chứng cần tạo trong dự án:

| Kiểm thử | Điều cần chứng minh |
| --- | --- |
| Unit test | Validation và logic trạng thái đơn giản đúng |
| Integration test với PostgreSQL | Constraint/transaction hoạt động đúng |
| Concurrent integration test | Không có `remaining_stock < 0`, accepted count đúng |
| RabbitMQ integration test | Event lặp không tạo order trùng |
| k6 load test | Throughput, latency, success/sold-out count dưới tải |

### Quy tắc bằng chứng cho portfolio

- Không mô tả một pattern là đã triển khai khi chưa có code và test cho đường lỗi chính.
- Không ghi số liệu hiệu năng hoặc tỉ lệ giảm tải nếu chưa có script, cấu hình
  chạy và report có thể tái tạo trong repository.
- CV có thể nói về mục tiêu thiết kế hiện tại; chỉ chuyển sang thành tựu sau
  khi test chứng minh invariant và event/order idempotency.

## 9. Câu hỏi phỏng vấn từ thiết kế này

**Tại sao không dùng một database chung cho tất cả service?**

Database riêng làm rõ ownership và giảm coupling. Đổi lại phải xử lý eventual
consistency khi dữ liệu đi qua event.

**Tại sao không trừ stock bằng cách đọc rồi save entity?**

Hai transaction có thể đọc cùng một giá trị trước khi ghi. Conditional atomic
update đưa phép kiểm tra và thay đổi thành một thao tác do database bảo vệ.

**Tại sao chưa dùng Redis làm stock chính?**

Tính đúng phải có trước tối ưu. PostgreSQL atomic update cho baseline dễ chứng
minh; Redis reservation nhanh hơn có thể được thử sau nhưng cần giải quyết
đồng bộ với database và message.

**Tại sao cần outbox?**

Database commit và RabbitMQ publish không nằm trong cùng transaction. Outbox
giữ event trong database cùng lúc với reservation, nhờ đó publisher có thể
retry mà không làm mất sự kiện.
