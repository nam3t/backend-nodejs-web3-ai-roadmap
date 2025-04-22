**Table of content:**

- [Advanced PostgreSQL](#advanced-postgresql)
  - [View \& Materialized View](#view--materialized-view)
    - [View](#view)
    - [Materialized View](#materialized-view)
    - [So sánh View và Materialized View](#so-sánh-view-và-materialized-view)
  - [indexes và tối ưu hóa query](#indexes-và-tối-ưu-hóa-query)
    - [Các loại index](#các-loại-index)
      - [B-tree (mặc định)](#b-tree-mặc-định)
      - [Hash](#hash)
      - [GIN (Generalized Inverted Index)](#gin-generalized-inverted-index)
      - [GiST (Generalized Search Tree)](#gist-generalized-search-tree)
      - [BRIN (Block Range Index)](#brin-block-range-index)
    - [Tối ưu query](#tối-ưu-query)
      - [phân tích query](#phân-tích-query)
      - [kỹ thuật tối ưu hoá](#kỹ-thuật-tối-ưu-hoá)
  - [triggers và stored procedures](#triggers-và-stored-procedures)
    - [triggers](#triggers)
    - [stored procedures](#stored-procedures)
    - [kết hợp trigger và stored procedure](#kết-hợp-trigger-và-stored-procedure)
  - [Window Functions và CTEs](#window-functions-và-ctes)
    - [Window Functions](#window-functions)
    - [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
  - [JSON/JSONB \& Partitioning](#jsonjsonb--partitioning)
    - [JSON/JSONB](#jsonjsonb)
    - [Partitioning](#partitioning)
- [Advanced MongoDB](#advanced-mongodb)

## Advanced PostgreSQL

### View & Materialized View

#### View

- là bảng ảo được tạo từ `SELECT`
- mỗi khi query view, Postgres sẽ chạy lại query gốc
- không lưu data vật lý => data luôn được update theo bảng gốc

```sql
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE is_active = true;
```

  => sử dụng `SELECT * FROM active_users;`

#### Materialized View

- lưu kết quả câu `SELECT` dưới dạng bảng vật lý
- data chỉ được update khi chạy `REFRESH MATERIALIZED VIEW`
- thích hợp các query phức tạp, cần hiệu suất cao và không thay đổi thường xuyên

```sql
CREATE MATERIALIZED VIEW user_order_summary AS
SELECT user_id, COUNT(*) AS total_orders
FROM orders
GROUP BY user_id;
```

  => query `SELECT * FROM user_order_summary;`
  => update data `REFRESH MATERIALIZED VIEW user_order_summary;`
  => để tránh khoá bảng khi refresh, sử dụng `CONCURRENTLY`: `REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_summary;`

#### So sánh View và Materialized View

|  | View | Materialized View |
| ----------- | ----------- |----------- |
| lưu trữ data | không | có |
| update data | tự động theo bảng gốc | thủ công bằng `REFRESH` |
| hiệu suất | phụ thuộc vào query gốc | nhanh hơn do data đã được lưu trữ |
| phù hợp với | data thay đổi thường xuyên | data ít thay đổi, query phức tạp |

### indexes và tối ưu hóa query

#### Các loại index

##### B-tree (mặc định)

- phù hợp các phép so sánh (`=`,`<`,`>`, `BETWEEN`)

  ```sql
  CREATE INDEX idx_users_email ON users(email);
  ```

##### Hash

- tối ưu cho phép so sánh `=`
- không hỗ trọ phép so sánh khác

```sql
CREATE INDEX idx_users_email_hash ON users USING hash(email);
```

##### GIN (Generalized Inverted Index)

- phù hợp với data là JSONB, array, text
- tốt cho tìm kiếm text hoặc query chứa nhiều value

```sql
CREATE INDEX idx_products_tags ON products USING GIN (tags);
```

##### GiST (Generalized Search Tree)

- hỗ trợ không gian, phạm vi
- thường được sử dụng với PostGIS cho data địa lý

```sql
CREATE INDEX idx_locations_geom ON locations USING GiST (geom);
```

##### BRIN (Block Range Index)

- tối ưu cho bảng lớn với data có thứ tự
- tiết kiệm không gian lưu trữ, phù hợp với data là thời gian

```sql
CREATE INDEX idx_logs_date ON logs USING BRIN (log_date);
```

#### Tối ưu query

##### phân tích query

- dùng `EXPLAIN` và `EXPLAIN ANALYZE`
- `EXPLAIN` hiển thị plan thực thi của query mà không thực hiện query

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

- `EXPLAIN ANALYZE` thực hiện query và show plan thực thi cùng thống kê thực tế

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
```

##### kỹ thuật tối ưu hoá

- sử dụng index phù hợp
  - tạo index cho các cột thường xuyên được sử dụng trong `WHERE`, `JOIN`, `ORDER BY`
  - tránh tạo index không cần thiết => gây chậm quá trình ghi data
- sử dụng partial index
  - tạo index cho một phần data thoả mãn điều kiện nhất định

```sql
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

- sử dụng expression index

```sql
CREATE INDEX idx_lower_email ON users(LOWER(email));
```

- tránh sử dụng `SELECT *`
- sử dụng `LIMIT` và `OFFSET`

```sql
SELECT * FROM users ORDER BY created_at DESC LIMIT 10 OFFSET 20;
```

### triggers và stored procedures

#### triggers

- là hàm tự động chạy khi xảy ra 1 event cụ thể (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`) trên một bảng hoặc view
- 2 loại trigger:
  - **row-level**: activate cho mỗi row bị ảnh hưởng
  - **statement-level**: activate một lần cho mỗi câu lệnh SQL, bất kể số row
- ví dụ ghi log khi update bảng `users`
  - tạo bảng log

  ```sql
  CREATE TABLE user_logs (
    id SERIAL PRIMARY KEY,
    user_id INT,
    action TEXT,
    log_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
  ```

  - tạo hàm trigger

  ```sql
  CREATE OR REPLACE FUNCTION log_user_update()
    RETURNS TRIGGER AS $$
    BEGIN
    INSERT INTO user_logs(user_id, action)
    VALUES (NEW.id, 'updated');
    RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
  ```

  - tạo trigger

  ```sql
  CREATE TRIGGER trg_log_user_update
    AFTER UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION log_user_update();
  ```
  
  => khi một row trong bảng `users` được update, hàm `log_user_update()` sẽ auto log thông tin vào bảng `user_logs`

#### stored procedures

- là tập hợp các câu lệnh SQL được lưu và chạy trên database server
- khác với `FUNCTION`, `PROCEDURE` không return value và có thể quản lý `transaction`
- ví dụ chuyển tiền giữa 2 tài khoản

```sql
// bảng accounts
CREATE TABLE accounts (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  balance DECIMAL(15,2) NOT NULL
);

INSERT INTO accounts(name, balance)
VALUES ('santi', 10000), ('itnas', 10000);
```

```sql
// tạo stored procedure
CREATE OR REPLACE PROCEDURE transfer(
  sender INT,
  receiver INT,
  amount DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
  -- Trừ tiền từ tài khoản người gửi
  UPDATE accounts
  SET balance = balance - amount
  WHERE id = sender;

  -- Cộng tiền vào tài khoản người nhận
  UPDATE accounts
  SET balance = balance + amount
  WHERE id = receiver;
END;
$$;
```

```sql
// call procedure
CALL transfer(1, 2, 1000);
```

#### kết hợp trigger và stored procedure

- trigger không thể trực tiếp gọi procedure nhưng có thể tạo 1 hàm trigger và trong đó dùng lệnh `CALL` để gọi procedure

```sql
// tạo hàm trigger
CREATE OR REPLACE FUNCTION trigger_call_procedure()
RETURNS TRIGGER AS $$
BEGIN
  CALL some_procedure(NEW.id);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```sql
// tạo trigger
CREATE TRIGGER trg_call_procedure
AFTER INSERT ON some_table
FOR EACH ROW
EXECUTE FUNCTION trigger_call_procedure();
```

### Window Functions và CTEs

#### Window Functions

- thực hiện tính toán trên một tập các row liên quan đến row hiện tại nhưng không các row lại
  => phù hợp tính toán rank, tổng tích luỹ, giá trị trước/sau...

```sql
function_name (arguments) OVER (
  [PARTITION BY partition_expression]
  [ORDER BY order_expression]
  [ROWS frame_specification]
)
```

- `PARTITION BY` chia data thành các nhóm
- `ORDER BY` xác định thứ tự trong mỗi partition
- `ROWS` xác định phạm vi (frame)

```sql
//  tính tổng tích luỹ `amount` cho mỗi `user_id` theo thứ tự `order_date`
SELECT
  user_id,
  order_date,
  amount,
  SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date) AS running_total
FROM orders;
```

```sql
// xếp hạng nhân viên trong từng phòng ban dựa trên `salary`
SELECT
  department,
  employee_name,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
FROM employees;
```

```sql
// lấy `salary` của nhân viên trước và sau trong danh sách sắp xếp theo `salary`
SELECT
  employee_name,
  salary,
  LAG(salary) OVER (ORDER BY salary) AS previous_salary,
  LEAD(salary) OVER (ORDER BY salary) AS next_salary
FROM employees;
```

#### Common Table Expressions (CTEs)

- định nghĩa tập kết quả tạm thời dùng làm tham chiếu trong query chính
  => cải thiện tính rõ ràng và dễ bảo trì, đặc biệt trong những query phức tạp hoặc đệ quy

```sql
WITH cte_name AS (
  SELECT ...
)
SELECT ...
FROM cte_name;
```

```sql
// sử dụng CTE để lọc các order trong 30 ngày và đếm số order
WITH recent_orders AS (
  SELECT * FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT customer_id, COUNT(*) AS order_count
FROM recent_orders
GROUP BY customer_id;
```

```sql
// sử dụng CTE đệ quy lấy tất cả nhân viên cấp dưới của các quản lý cấp cao nhất
WITH RECURSIVE subordinates AS (
  SELECT employee_id, manager_id, employee_name
  FROM employees
  WHERE manager_id IS NULL
  UNION ALL
  SELECT e.employee_id, e.manager_id, e.employee_name
  FROM employees e
  INNER JOIN subordinates s ON e.manager_id = s.employee_id
)
SELECT * FROM subordinates;
```

### JSON/JSONB & Partitioning

#### JSON/JSONB

- `JSON` lưu data dưới dạng text, giữ nguyên format ban đầu
- `JSONB` lưu data dưới dạng binary, tối ưu cho query và indexing
  => nên sử dụng `JSONB` trong đa số trường hợp bì hiệu suất và khả năng tạo index tốt

```sql
// tạo bảng với column JSONB
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  details JSONB
);

// insert data
INSERT INTO products (name, details)
VALUES
  ('Laptop', '{"brand": "Dell", "specs": {"cpu": "i7", "ram": "16GB"}}'),
  ('Smartphone', '{"brand": "Samsung", "specs": {"cpu": "Exynos", "ram": "8GB"}}');

// query value của key 
  SELECT details->>'brand' AS brand FROM products;

// filter theo value 
  SELECT * FROM products WHERE details->'specs'->>'ram' = '16GB';

// update value
  UPDATE products
  SET details = jsonb_set(details, '{specs,ram}', '"32GB"')
  WHERE id = 1;

/// delete key
  UPDATE products
  SET details = details - 'specs'
  WHERE id = 1;
```

#### Partitioning

- là kỹ thuật chia nhỏ bảng lớn thành các bảng con (partition) dựa trên key partition, giúp tối ưu hiệu suất query và quản lý data
- các phương pháp partitioning:
  - **Range**: chia data dựa trên phạm vi giá trị (ví dụ: ngày tháng)
  - **List**: chia data dựa trên danh sách các giá trị cụ thể
  - **Hash**: chia data dựa trên giá trị hash của key partition
- lợi ích:
  - tăng hiệu suất query khi chỉ quét một partition thay vì toàn bộ bảng
  - dễ bảo trì như xoá hoặc sao lưu các partition cũ
  - hỗ trợ phân phối data trên các thiết bị lưu trữ khác nhau

```sql
// tạo partition
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  order_date DATE NOT NULL,
  amount DECIMAL
) PARTITION BY RANGE (order_date);

// tạo partition cụ thể
CREATE TABLE orders_2023 PARTITION OF orders
  FOR VALUES FROM ('2023-01-01') TO ('2023-12-31');

CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-12-31');
```

- ví dụ thực tế: quản lý đơn hàng

```sql
// tạo bảng order
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  order_date DATE NOT NULL,
  customer_id INT NOT NULL,
  order_details JSONB
) PARTITION BY RANGE (order_date);

// tạo partition theo năm
CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-12-31');

CREATE TABLE orders_2025 PARTITION OF orders
  FOR VALUES FROM ('2025-01-01') TO ('2025-12-31');

// insert data
INSERT INTO orders (order_date, customer_id, order_details)
VALUES
  ('2024-06-15', 1, '{"items": [{"product": "Laptop", "quantity": 1}], "total": 1200}'::jsonb),
  ('2025-03-22', 2, '{"items": [{"product": "Smartphone", "quantity": 2}], "total": 1500}'::jsonb);

// query order trong năm 2024
  SELECT * FROM orders
  WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

// query tổng số tiền của order cụ thể
  SELECT order_details->>'total' AS total_amount
  FROM orders
  WHERE id = 1;
```

## Advanced MongoDB

