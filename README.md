# Load Testing repo

Repo này chứa các scripts phục vụ việc mô phỏng tải lên hệ thống [Mini ecommerce](https://github.com/NT114-Q21-Specialized-Project/mini-ecommerce-microservices).

Hiện tại repo giữ các script chính:

- `scripts-k6/user-journey.js`
- `scripts/run-user-journey.sh`
- `scripts/setup/*`
- `scripts/user-service/*`
- `scripts/product-service/*`

## Flow chính

Flow hiện tại mô phỏng một customer đơn giản:

1. `setup()` login một hoặc nhiều customer trước khi bắt đầu test.
2. Trong mỗi iteration, gọi `GET /api/v1/products` để lấy danh sách sản phẩm.
3. Chọn ngẫu nhiên một sản phẩm rồi gọi `GET /api/v1/products/{id}` để xem chi tiết.
4. Gọi `GET /api/v1/orders` để xem danh sách đơn hàng của customer hiện tại.

Các API đang bị hit:

- `POST /api/v1/users/login`
- `GET /api/v1/products`
- `GET /api/v1/products/{id}`
- `GET /api/v1/orders`

Các request đều đi qua `api-gateway`, sau đó route vào:

- `user-service` cho `POST /api/v1/users/login`
- `product-service` cho `GET /api/v1/products`
- `product-service` cho `GET /api/v1/products/{id}`
- `order-service` cho `GET /api/v1/orders`

## Run script mô phỏng

Trước khi chạy, hãy tạo file env local từ file mẫu:

- `scripts-k6/.env.k6.example`

Ví dụ:

```bash
cd scripts-k6
cp .env.k6.example .env.k6
```

Sau đó cấu hình các biến môi trường trong:

- `scripts-k6/.env.k6`

**Mục tiêu**

- **`BASE_URL`**: địa chỉ hệ thống `mini-ecommerce` để k6 bắn tải vào.
- **`LOGIN_ENDPOINT`**: endpoint đăng nhập để lấy token.
- **`PRODUCTS_ENDPOINT`**: endpoint danh sách và chi tiết sản phẩm.
- **`ORDERS_ENDPOINT`**: endpoint danh sách đơn hàng.

**Điều khiển tải theo VU đơn giản**

- **`VU_START`**: số VU khởi đầu.
- **`VU_MAX`**: số VU tối đa.
- **`VU_STEP`**: số VU tăng thêm mỗi chu kỳ.
- **`VU_TIME_UNIT`**: khoảng thời gian của mỗi lần tăng VU.
- **`VU_RAMP_DOWN_DURATION`**: thời gian hạ tải về `0`.

**Xác thực**

- **`CUSTOMER_EMAIL`**: email customer khi chỉ chạy một user.
- **`CUSTOMER_PASSWORD`**: mật khẩu customer dùng để login.
- **`AUTH_TOKEN`**: token có thể điền tay nếu muốn bỏ qua login khi chỉ chạy một user.
- **`CUSTOMER_COUNT`**: số lượng customer muốn dùng.
- **`CUSTOMER_EMAIL_PREFIX`**: prefix email khi muốn sinh nhiều customer theo mẫu.
- **`CUSTOMER_EMAIL_DOMAIN`**: domain email cho nhóm customer sinh tự động.
- **`CUSTOMER_EMAILS`**: danh sách email phân tách bằng dấu phẩy nếu muốn chỉ định thủ công nhiều customer.

**Wrapper một lệnh**

- **`SEED_CUSTOMERS_BEFORE_RUN`**: tự tạo customer test trước khi chạy.
- **`CLEANUP_CUSTOMERS_AFTER_RUN`**: tự soft-delete customer test sau khi chạy xong.
- **`SEED_PRODUCTS_BEFORE_RUN`**: seed thêm product trước khi chạy nếu catalog đang thiếu dữ liệu.
- **`CLEANUP_PRODUCTS_AFTER_RUN`**: cleanup product và seller seed sau khi chạy xong.

Khuyến nghị mặc định cho repo hiện tại:

```text
SEED_CUSTOMERS_BEFORE_RUN=true
CLEANUP_CUSTOMERS_AFTER_RUN=true
SEED_PRODUCTS_BEFORE_RUN=true
CLEANUP_PRODUCTS_AFTER_RUN=true
```

Ví dụ nhiều customer:

```text
CUSTOMER_COUNT=20
CUSTOMER_EMAIL_PREFIX=k6.customer
CUSTOMER_EMAIL_DOMAIN=example.test
CUSTOMER_PASSWORD=K6Read@12345
```

Khi đó script sẽ login các user:

- `k6.customer.1@example.test`
- `k6.customer.2@example.test`
- ...
- `k6.customer.20@example.test`

**Flow đơn giản**

- **`CATALOG_PAGE_SIZE`**: số lượng sản phẩm lấy ở mỗi lần gọi list.
- **`THINK_TIME_MIN`**: thời gian nghỉ tối thiểu giữa các bước.
- **`THINK_TIME_MAX`**: thời gian nghỉ tối đa giữa các bước.
- **`REQUEST_TIMEOUT`**: timeout cho mỗi HTTP request.

Chạy trọn flow một lệnh:

```bash
cd /home/tienphatng237/Desktop/NT114-Q21-Speacialized-Project/load-testing
K6_WEB_DASHBOARD=true bash ./scripts/run-user-journey.sh
```

Flow wrapper:

1. Seed customer test theo cấu hình trong `.env.k6`
2. Chạy `scripts-k6/user-journey.js`
3. Cleanup customer test sau khi k6 kết thúc
4. Cleanup product/seller seed nếu có bật seed product trước đó

Khi `CUSTOMER_COUNT > 1`, wrapper sẽ tự thêm `RUN_ID` vào `CUSTOMER_EMAIL_PREFIX` để mỗi lần chạy dùng một nhóm customer riêng.

Khi có seed product, wrapper cũng tự thêm `RUN_ID` vào `SEED_NAMESPACE` để seller và product seed của từng lần chạy không đụng nhau.

## Seed dữ liệu catalog

Khi catalog đã hết hàng, có thể seed nhanh seller và product mới trước khi chạy tải:

```bash
bash ./scripts/product-service/seed-products.sh
```

Dọn seller seed sau khi test:

```bash
bash ./scripts/product-service/cleanup-seed-users.sh
```

Lưu ý:

- cleanup sẽ xóa product seed theo tên đã tạo, rồi mới soft-delete seller seed
- flow này giả định `product-service` đã bật CRUD `DELETE /api/v1/products/{id}`
