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
  - [Aggregation Pipeline](#aggregation-pipeline)
    - [các stage nâng cao](#các-stage-nâng-cao)
      - [`$lookup`](#lookup)
      - [`$facet`](#facet)
      - [`$graphLookup`](#graphlookup)
      - [`$function`](#function)
  - [Tối ưu query](#tối-ưu-query-1)
    - [phân tích query với `explain()`](#phân-tích-query-với-explain)
    - [index](#index)
    - [projection](#projection)
    - [phân trang với `limit()` và `skip()`](#phân-trang-với-limit-và-skip)
    - [`hint()`](#hint)
  - [schema](#schema)
    - [Embedding vs. Referencing](#embedding-vs-referencing)
      - [embedding](#embedding)
      - [referencing](#referencing)
    - [design pattern](#design-pattern)
      - [Extended reference pattern](#extended-reference-pattern)
      - [polymorphic pattern](#polymorphic-pattern)
      - [versioning pattern](#versioning-pattern)
      - [bucket pattern](#bucket-pattern)
  - [Transactions](#transactions)
  - [Replication và Sharding](#replication-và-sharding)
    - [Replication](#replication)
      - [config Replica Set](#config-replica-set)
    - [Sharding](#sharding)
      - [Shard Cluster architecture](#shard-cluster-architecture)

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

### Aggregation Pipeline

- là chuỗi các stage xử lý data, mỗi stage thực hiện một phép biến đổi cụ thể trên input data và truyền output kết quả cho stage tiếp theo
- các stage phổ biến:
  - `$match` lọc document theo điều kiện
  - `$group` nhóm document và thực hiện các phép toán như tính tổng, trung bình
  - `$project` chọn và format lại các field
  - `$sort` sắp xếp document

#### các stage nâng cao

##### `$lookup` 

- cho phép kết hợp data giữa các collection, tương tự `JOIN` trong SQL

```mongodb
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerDetails"
    }
  }
]);
```

##### `$facet`

- cho phép chạy nhiều pipeline song song và trả về kết quả tổng hợp

```mongodb
db.products.aggregate([
  {
    $facet: {
      "priceStats": [
        { $group: { _id: null, avgPrice: { $avg: "$price" } } }
      ],
      "categoryCounts": [
        { $group: { _id: "$category", count: { $sum: 1 } } }
      ]
    }
  }
]);
```

##### `$graphLookup`

- thực hiện tìm kiếm đệ quy trong một collection, phù hợp data dạng cây hoặc đồ thị

```mongodb
db.employees.aggregate([
  {
    $graphLookup: {
      from: "employees",
      startWith: "$managerId",
      connectFromField: "managerId",
      connectToField: "_id",
      as: "managementHierarchy"
    }
  }
]);
```

##### `$function`

- viết hàm js tuỳ chỉnh để xử lý data trong pipeline

```mongodb
db.collection.aggregate([
  {
    $addFields: {
      computedField: {
        $function: {
          body: function(value) {
            return value.toUpperCase();
          },
          args: ["$fieldName"],
          lang: "js"
        }
      }
    }
  }
]);
```

### Tối ưu query

#### phân tích query với `explain()`

- `explain()` cung cấp thông tin chi tiết cách mongodb thực hiện query:
  - plan query
  - số lượng document scan
  - index được sử dụng

```mongodb
db.users.find({ age: { $gte: 30 } }).explain("executionStats");

// Kết quả sẽ hiển thị các thông tin như totalDocsExamined, totalKeysExamined, và executionTimeMillis
```

#### index

- index giúp mongodb query nhanh hơn

```mongodb
// index đơn giản
db.users.createIndex({ age: 1 });

// index tổng hợp
db.users.createIndex({ age: 1, name: 1 }); // nên tuần theo quy tắc ESR (Equality, Sort, Range)
```

- tránh tạo quá nhiều index, mỗi index đều tốn memory và ảnh hưởng đến việc write data

```mongodb
// kiểm tra việc sử dụng index
db.users.aggregate([{ $indexStats: {} }]);
```

#### projection

- projection cho phép chỉ lấy những data cần thiết, giảm dung lượng truyền tải và tăng hiệu suất query

```mongodb
db.users.find({ age: { $gte: 30 } }, { name: 1, email: 1, _id: 0 });
```

#### phân trang với `limit()` và `skip()`

```mongodb
db.users.find().skip(20).limit(10);
```

- lưu ý khi sử dụng `skip()` với giá trị cao có thể ảnh hưởng đến hiệu suất. Trong trường hợp này có thể sử dụng phân trang dựa trên con trỏ (cursor-based pagination)

#### `hint()`

- chỉ định mongodb sử dụng một số index cụ thể

```mongodb
db.users.find({ age: { $gte: 30 } }).hint({ age: 1 });
```

### schema

#### Embedding vs. Referencing

##### embedding

- embedding là việc lưu data liên quan trong cùng một document, giúp query nhanh hơn và tính toàn vẹn của data
- sử dụng embedding khi:
  - quan hệ 1:1 hoặc 1:N với số lượng N nhỏ
  - data liên quan được query cùng nhau thường xuyên
  - data liên quan không thay đổi thường xuyên

```mongodb
{
  "_id": ObjectId("..."),
  "name": "santi",
  "orders": [
    { "order_id": 1001, "item": "Laptop", "price": 1200 },
    { "order_id": 1002, "item": "Mouse", "price": 25 }
  ]
}
```

##### referencing

- referencing là việc lưu data liên quan ở các collection riêng biệt và sử dụng tham chiếu để liên kết, giúp tránh lặp và dễ quản lý data lớn
- sử dụng referencing khi:
  - quan hệ 1:N với N lớn hoặc N:M
  - data liên quan được query độc lập
  - data liên quan thay đổi thường xuyên

```mongodb
// Collection: customers
{
  "_id": ObjectId("..."),
  "name": "santi"
}

// Collection: orders
{
  "_id": ObjectId("..."),
  "customer_id": ObjectId("..."),
  "item": "Laptop",
  "price": 1200
}
```

#### design pattern

##### Extended reference pattern

- kết hợp giữa embedding và referencing bằng cách embed một phần data được tham chiếu để giảm số lần query

```mongodb
// hiển thị danh sách bạn bè mà không cần query thêm
{
  "_id": ObjectId("..."),
  "name": "santi",
  "friends": [
    {
      "_id": ObjectId("..."),
      "name": "itnas",
      "profilePic": "..."
    },
    ...
  ]
}
```

##### polymorphic pattern

- lưu các loại document khác nhau trong cùng một collection bằng field phân biệt loại, giúp đơn giản hoá việc quản lý và query các loại data khác nhau

```mongodb
{
  "_id": ObjectId("..."),
  "type": "image",
  "url": "..."
}

{
  "_id": ObjectId("..."),
  "type": "video",
  "url": "...",
  "duration": 120
}
```

##### versioning pattern

- lưu các version khác nhau của document để theo dõi thay đổi, phù hợp các app cần theo dõi thay đổi hoặc undo/redo

```mongodb
{
  "_id": ObjectId("..."),
  "document_id": ObjectId("..."),
  "version": 1,
  "content": "..."
}

{
  "_id": ObjectId("..."),
  "document_id": ObjectId("..."),
  "version": 2,
  "content": "..."
}
```

##### bucket pattern

- nhóm nhiều document nhỏ vào một document lớn hơn để giảm số lượng document và tăng hiệu suất, thường dùng trong app IoT hoặc ghi nhận data theo thời gian

```mongodb
{
  "_id": ObjectId("..."),
  "sensor_id": "sensor_1",
  "readings": [
    { "timestamp": ISODate("2025-04-23T00:00:00Z"), "value": 23.5 },
    { "timestamp": ISODate("2025-04-23T00:01:00Z"), "value": 24.0 },
    ...
  ]
}
```

### Transactions

- yêu cầu:
  - mongodb version 4.0 trở lên
  - kết nối đến một replica set hoặc shared cluster
- transaction trong mongodb tương tự trong SQL, thực hiện một chuỗi các thao tác read/write trong một transaction duy nhất
- nếu bất kỳ thao tác nào thất bại, toàn bộ transaction bị huỷ => đảm bảo data consistent
- cần handle các lỗi có thể xảy ra như `TransientTransactionError` hoặc `UnknownTransactionCommitResult`
  
```mongodb
// dùng core API NodeJS
async function transferFunds(client) {
  const session = client.startSession();
  try {
    session.startTransaction();

    const accounts = client.db('bank').collection('accounts');

    await accounts.updateOne(
      { accountId: 'A123' },
      { $inc: { balance: -100 } },
      { session }
    );

    await accounts.updateOne(
      { accountId: 'B456' },
      { $inc: { balance: 100 } },
      { session }
    );

    await session.commitTransaction();
    console.log('Giao dịch thành công!');
  } catch (error) {
    console.error('Lỗi giao dịch:', error);
    await session.abortTransaction();
  } finally {
    await session.endSession();
  }
}


// dùng Convenient Transaction API
async function transferFunds(client) {
  await client.withSession(async (session) => {
    await session.withTransaction(async () => {
      const accounts = client.db('bank').collection('accounts');

      await accounts.updateOne(
        { accountId: 'A123' },
        { $inc: { balance: -100 } },
        { session }
      );

      await accounts.updateOne(
        { accountId: 'B456' },
        { $inc: { balance: 100 } },
        { session }
      );
    });
  });
}
```

### Replication và Sharding

#### Replication

- Replica Set là một nhóm các server mongodb duy trì cùng một tập data:
  - high availability: nếu một node lỗi, node khác sẽ sẵn sàng thay thế
  - an toàn: data được copy sang nhiều node
  - có thể phân phối tải đọc giữa các node thứ cấp

##### config Replica Set

- bước 1: cài mongo trên ít nhất 3 server
- bước 2: config file `mongod.conf` trên server:

  ```yaml
  replication:
  replSetName: "rs0"
  ```

- bước 3: khởi động mongodb trên mỗi server:
  
  ```bash
  mongod --config /etc/mongod.conf
  ```

- bước 4: init Replica Set từ một node:
  
  ```mongodb
    rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "mongo1:27017" },
        { _id: 1, host: "mongo2:27017" },
        { _id: 2, host: "mongo3:27017" }
    ]
    })
  ```

- bước 5: check status
  
  ```mongodb
  rs.status()
  ```

- lưu ý:
  - arbiter: nếu chỉ có 2 node, thêm một arbiter để đảm bảo số vote lẻ
  - oplog: theo dõi và điều chỉnh oplog size cho phù hợp với tải ghi của hệ thống
  - bảo mật: sử dụng `keyFile` hoặc X.509 certificate để xác thực  giữa các node

#### Sharding

- là kỹ thuật chia data thành các phần nhỏ (shard) và phân phối trên nhiều server
  => scale horizontal, xử lý khối lượng data lớn
  => tăng performance do phân tán tải query và write data
  => tăng khả năng chịu lỗi, mỗi shard có thể là một replica set

##### Shard Cluster architecture

- Shard: lưu data thực tế
- Config Server: lưu metadata và phân phối data
- mongos (query router): routing query đến shard phù hợp

- cách triển khai:
  - bước 1: start config server replica set

  ```bash
  mongod --configsvr --replSet configReplSet --dbpath /data/configdb --port 27019 --bind_ip localhost
  ```

  - bước 2: init config server replica set

  ```mongodb
  rs.initiate({
    _id: "configReplSet",
    configsvr: true,
    members: [
        { _id: 0, host: "cfg1:27019" },
        { _id: 1, host: "cfg2:27019" },
        { _id: 2, host: "cfg3:27019" }
    ]
    })
  ```

  - bước 3: start shard server (mỗi shard là một replica set)

  ```bash
  mongod --shardsvr --replSet shardReplSet1 --dbpath /data/shard1 --port 27018 --bind_ip localhost
  ```

  - bước 4: init shard replica set

  ```mongodb
  rs.initiate({
    _id: "shardReplSet1",
    members: [
        { _id: 0, host: "shard1a:27018" },
        { _id: 1, host: "shard1b:27018" },
        { _id: 2, host: "shard1c:27018" }
    ]
    })
  ```

  - bước 5: start mongos và kết nối với config server

  ```bash
  mongos --configdb configReplSet/cfg1:27019,cfg2:27019,cfg3:27019 --bind_ip localhost
  ```

  - bước 6: thêm shard vào cluster

  ```mongodb
  sh.addShard("shardReplSet1/shard1a:27018,shard1b:27018,shard1c:27018")
  sh.addShard("shardReplSet2/shard2a:27018,shard2b:27018,shard2c:27018")
  ```

  - bước 7: kích hoạt sharding cho collection cụ thể

  ```mongodb
  use my_db

  sh.enableSharding("my_db")

  // Sharding theo Hash (phân phối dữ liệu đồng đều)
  sh.shardCollection("my_db.my_collection", { "my_shard_key": "hashed" })

  // Sharding theo Range (phù hợp với query theo phạm vi)
  sh.shardCollection("my_db.my_collection", { "my_shard_key": 1 })
  ```

  - bước 8: check status sharding

  ```mongodb
  sh.status()
  // hiển thị thông tin về các Shard, db đã được kích hoạt Sharding và các collection liên quan
  ```

  - bước 9: check phân phối data

  ```mongodb
  db.your_collection.getShardDistribution()
  // hiển thị số lượng document và data size trên từng Shard
  ```
