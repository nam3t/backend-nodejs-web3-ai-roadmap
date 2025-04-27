**Table of content:**

- [Design patterns](#design-patterns)
  - [Singleton](#singleton)
  - [Factory](#factory)
  - [Builder](#builder)
  - [Observer](#observer)
  - [Strategy](#strategy)
  - [Dependency Injection (DI)](#dependency-injection-di)
  - [Repository](#repository)
  - [Adapter (Wrapper)](#adapter-wrapper)
  - [Decorator](#decorator)
  - [Proxy](#proxy)
- [Best Practice](#best-practice)
  - [tách biệt layer](#tách-biệt-layer)
  - [quản lý config tập trung](#quản-lý-config-tập-trung)
  - [logging](#logging)
  - [error handling](#error-handling)
  - [testing](#testing)
  - [CI/CD](#cicd)

## Design patterns

### Singleton

- là pattern thuộc nhóm **Creational Pattern**
- đảm bảo rằng một class chỉ có duy nhất 1 instance trong cả app
- có thể access global đến instance đó
- **thường được dùng khi cần instance dùng chung trên toàn hệ thống, vd như: db connection, logger.. (những loại chỉ nên có 1 instance duy nhất để tránh conflict)**
- chỉ dùng khi cần thiết vì lạm dụng có thể tạo những biến global ẩn gây khó khăn cho việc test

``` ts
// ví dụ implement singleton Logger
// mỗi khi call getInstance() sẽ luôn trả về obj Logger tạo đầu tiên
// đảm bảo mọi nơi trong app dùng chung một logger
class Logger {
  private static instance: Logger;

  private constructor() {}  // constructor private để ngăn tạo mới từ bên ngoài

  static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  log(message: string): void {
    console.log(`[LOG]: ${message}`);
  }
}

// Sử dụng
const logger1 = Logger.getInstance();
const logger2 = Logger.getInstance();
logger1.log("Node.js Singleton Logger");  // [LOG]: Node.js Singleton Logger
console.log(logger1 === logger2);  // true, cả hai đều là cùng một instance
```

### Factory

- thuộc nhóm **Creational Pattern**
- dùng để tạo các obj mà không cần chỉ rõ class của nó khi tạo
- **thay vì dùng trực tiếp `new`, ta đưa logic khởi tạo obj vào một Factory, qua đó đóng gói việc tạo obj**
- giúp code linh hoạt hơn khi cần mở rộng và cần thay đổi cách khởi tạo obj
- có nhiều biến thể như:
  - **Simple Factory**: một hàm tạo obj đơn giản
  - **Factory Method**: định nghĩa ở class cha, cho phép class con quyết định obj được tạo
  - **Abstract Factory**: cung cấp interface để tạo những obj liên quan
- **về cơ bản, nên sử dụng **Factory** khi việc khởi tạo phức tạp, phụ thuộc vào điều kiện cụ thể**

```ts
// Simple Factory tạo obj hình học theo tham số đầu vào
// interface Shape, và các class Circle, Square
// Factory có static method createShape(type) để trả về obj phù hợp

interface Shape {
  draw(): void;
}

class Circle implements Shape {
  draw(): void {
    console.log("Vẽ hình tròn");
  }
}

class Square implements Shape {
  draw(): void {
    console.log("Vẽ hình vuông");
  }
}

class ShapeFactory {
  static createShape(type: string): Shape {
    switch(type) {
      case "circle":
        return new Circle();
      case "square":
        return new Square();
      default:
        throw new Error("Loại shape không được hỗ trợ");
    }
  }
}

// Sử dụng Factory
const shape1 = ShapeFactory.createShape("circle");
shape1.draw();  // Vẽ hình tròn

const shape2 = ShapeFactory.createShape("square");
shape2.draw();  // Vẽ hình vuông
```

- `ShapeFactory` là Factory quyết định trả về `Circle` hay `Square` dựa trên tham số đầu vào
- qua Factory ta sẽ không cần biết class nào được tạo, ví dụ có thể thêm class `Triangle` mà không cần thay đổi code khi sử dụng, chỉ cần mở rộng Factory
- Pattern này phù hợp khi có nhiều subclass và logic chọn class phức tạp, tránh lặp nhiều `if/else`

### Builder

- cũng là **Creational Pattern**
- dùng để từng bước tạo ra obj phức tạp
- **thay vì dùng constructor với quá nhiều tham số hoặc dùng nhiều hàm `factory`, Builder tách quá trình tạo obj thành nhiều bước nhỏ qua một obj _builder_**
- giúp tạo obj linh hoạt, dễ đọc và **có thể tạo các biến thể khác nhau** của obj với cùng quy trình
- **phù hợp với các obj có nhiều option property hoặc cần đi qua nhiều bước khởi tạo trước khi sẵn sàng sử dụng (vd: config connection, load data, verify...)**

```ts
// Query Builder cho câu lệnh SQL
// Builder cho phép chọn bảng (from), chọn cột (select), thêm điều kiện (where) và cuối cùng là build() để tạo ra query hoàn chỉnh
// Mỗi bước sẽ tạo ra chính obj builder để có thể gọi liên tiếp (chaining)

class QueryBuilder {
  private table: string = "";
  private fields: string[] = [];
  private condition: string = "";

  select(fields: string[]): QueryBuilder {
    this.fields = fields;
    return this;
  }

  from(table: string): QueryBuilder {
    this.table = table;
    return this;
  }

  where(condition: string): QueryBuilder {
    this.condition = condition;
    return this;
  }

  build(): string {
    const fieldsStr = this.fields.length ? this.fields.join(", ") : "*";
    let query = `SELECT ${fieldsStr} FROM ${this.table}`;
    if (this.condition) {
      query += ` WHERE ${this.condition}`;
    }
    return query;
  }
}

// Sử dụng
const query = new QueryBuilder()
  .select(["name", "age"])
  .from("users")
  .where("age > 18")
  .build();
console.log(query);  
// Output: SELECT name, age FROM users WHERE age > 18
```

- cách sử dụng của builder cho thấy code dễ đọc, dễ add/remove các component của query
- đảm bảo tính độc lập giữa quá trình build và kết quả cuối cùng
- có thể dễ dàng thêm bước (vd: `GROUP BY`, `ORDER BY`) mà không ảnh hưởng cách sử dụng

### Observer

- là **Behavioral Pattern**
- định nghĩa cơ chế sub/unsub (pub/sub) để một obj (Subject) có thể thông báo đến nhiều obj khác (Observer) khi có event hoặc status change
- tạo ra mối quan hệ một-nhiều, 1 subject quản lý danh sách các observers, khi subject có sự thay đổi, nó notify cho tất cả observers
- **phù hợp khi cần tách biệt nguồn phát event và xử lý event**

```ts
// EventEmitter trong NodeJS
// EventEmitter là Subject, listener là Observer

import { EventEmitter } from 'events';
const eventBus = new EventEmitter();

// Đăng ký hai Observer lắng nghe sự kiện "orderCreated"
eventBus.on('orderCreated', (order) => {
  console.log("Observer 1: New order:", order);
});
eventBus.on('orderCreated', (order) => {
  console.log("Observer 2: Logging order", order);
});

// Khi một đơn hàng mới được tạo, Subject phát sự kiện cho tất cả observers
const newOrder = { id: 101, item: "Sách Node.js", qty: 1 };
eventBus.emit('orderCreated', newOrder);

// Console output:
// Observer 1: New order: { id: 101, item: 'Sách Node.js', qty: 1 }
// Observer 2: Logging order { id: 101, item: 'Sách Node.js', qty: 1 }
```

### Strategy

- là **Behavioral Pattern**
- định nghĩa những thuật toán hoặc hành vi khác nhau, đóng gói chúng vào một class riêng và làm cho chúng có thể thay thế nhau dễ dàng
- thường bao gồm: 
  - **Strategy Interface**: định nghĩa method chung cho thuật toán
  - **nhiều Concrete Strategy**: các implementation của những thuật toán đó
  - **Context**: để sử dụng Strategy
- tại runtime sẽ chọn strategy thích hợp để thực thi, giúp tránh viết nhiều câu lệnh điều kiện chọn thuật toán
- **nên dùng khi có nhiều strategy khác nhau để hoàn thành 1 công việc và có thể thay đổi quyết định chọn strategy nào (theo config, data type, runtime...)**

```ts
// sorting system
// interface SortStrategy và 2 thuật toán cụ thể QuickSort và BubbleSort
// Sorter là Context chứa 1 Strategy và dùng nó để sort

interface SortStrategy {
  sort(data: number[]): number[];
}

class QuickSortStrategy implements SortStrategy {
  sort(data: number[]): number[] {
    console.log("Sử dụng QuickSort");
    // (Giả sử ở đây triển khai thuật toán QuickSort)
    return data.sort((a, b) => a - b);
  }
}

class BubbleSortStrategy implements SortStrategy {
  sort(data: number[]): number[] {
    console.log("Sử dụng BubbleSort");
    // (Triển khai Bubble Sort minh họa)
    for (let i = 0; i < data.length; i++) {
      for (let j = 0; j < data.length - 1; j++) {
        if (data[j] > data[j+1]) {
          [data[j], data[j+1]] = [data[j+1], data[j]];
        }
      }
    }
    return data;
  }
}

class Sorter {
  constructor(private strategy: SortStrategy) {}
  setStrategy(strategy: SortStrategy) {
    this.strategy = strategy;
  }
  sortData(data: number[]): number[] {
    return this.strategy.sort(data);
  }
}

// Sử dụng
const dataset = [5, 1, 4, 3, 2];
const sorter = new Sorter(new QuickSortStrategy());
let result = sorter.sortData([...dataset]);  
// Output console: "Sử dụng QuickSort"
console.log(result);  // [1, 2, 3, 4, 5]

sorter.setStrategy(new BubbleSortStrategy());
result = sorter.sortData([...dataset]);  
// Output console: "Sử dụng BubbleSort"
console.log(result);  // [1, 2, 3, 4, 5]
```

### Dependency Injection (DI)

- không thuộc "Gang of Four" design pattern
- là kỹ thuật thiết kế quan trọng thuộc nguyên lý **Inversion of Control**
- **thay vì một obj tự khởi tạo hoặc tự tìm các dependency của nó, việc đó sẽ được inject vào từ bên ngoài**
- **DI cho phép truyền vào các dependency mà 1 obj cần, thường thông qua constructor hoặc setter, do một container hoặc component khác cung cấp**
- **mục đích là tách rời sự phụ thuộc giữa các components**

```ts
// ví dụ có UserService phụ thuộc UserRepository
// thay vì UserService tự tạo `new UserRepository()`, ta inject từ ngoài vào

// Giả sử interface và triển khai của Repository
interface IUserRepository {
  findById(id: string): Promise<User|null>;
}
class UserRepository implements IUserRepository {
  async findById(id: string): Promise<User|null> {
    // logic truy vấn DB để tìm user
    return { id, name: "Temp User" };  // dữ liệu giả
  }
}

// Service phụ thuộc Repository
class UserService {
  constructor(private userRepo: IUserRepository) {}  // inject qua constructor

  async getUserProfile(id: string) {
    const user = await this.userRepo.findById(id);
    // áp dụng thêm logic nghiệp vụ nếu cần
    return user;
  }
}

// Sử dụng: khởi tạo Service và inject Repository vào
const repo = new UserRepository();
const userService = new UserService(repo);
userService.getUserProfile("123").then(console.log);
```

- NestJS là ví dụ điển hình của DI trong Node. Mỗi Service, Repository đều được DI quản lý, code chỉ cần khai báo dependency trong constructor, Nest sẽ tự resolve và tạo instance thích hợp

### Repository

- thường dùng ở data layer của app
- pattern này tạo ra class trung gian giữa **business logic** và **data source**
- chịu trách nhiệm CRUD và cung cấp business method để thao tác với data (vd: `findUserbyEmail`, `getActiveOrders`..)
- **mục tiêu là tách biệt logic access data với business logic**
- thường đi kèm với **Unit of Work** ở các app phức tạp

```ts
// Repository cho bảng User
interface User {
  id: string;
  name: string;
  email: string;
}

interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findAll(): Promise<User[]>;
  save(user: User): Promise<void>;
}

class MongoUserRepository implements IUserRepository {
  // Giả sử có kết nối MongoDB sẵn, có thể inject qua constructor (DI)
  async findById(id: string): Promise<User | null> {
    // Thay vì trả về trực tiếp từ ORM, ta có thể xử lý thành đối tượng User
    console.log("MongoRepo: find user by id", id);
    // ... code gọi MongoDB
    return { id, name: "Temp", email: "temp@example.com" };  // dữ liệu mẫu
  }
  async findAll(): Promise<User[]> {
    console.log("MongoRepo: find all users");
    // ... code lấy tất cả user
    return [];
  }
  async save(user: User): Promise<void> {
    console.log("MongoRepo: save user", user);
    // ... code thêm hoặc cập nhật user trong DB
  }
}
```

- `MongoUserRepository` đóng gói chi tiết thao tác với mongodb, class Service có thể dùng interface `IUserRepository` thay vì phụ thuộc vào mongodb
- thực tế, nếu dùng ORM như TypeORM, pattern Repository đã được tích hợp
- layered architecture (Controller-Service-Repository) trong app của NestJS thường là Controller handle HTTP req, gọi Service để thực hiện business logic, Service sẽ gọi Repository để làm việc với db
  => Controller không động vào SQL, Service không biết chi tiết SQL, Repository không chứa logic trình bày

### Adapter (Wrapper)

- thuộc nhóm **Structural Pattern**
- **cho phép 2 class có interface không tương thích có thể làm việc cùng nhau bằng cách thêm class trung gian là Adapter**
- Adapter biến đổi interface của 1 class (Adaptee) thành interface mà client mong đợi (Target)
- **được dùng khi muốn tái sử dụng một đoạn code hiện có nhưng interface của nó lại không tương thích với hệ thống hiện tại hoặc dùng khi muốn tích hợp 1 lib từ bên ngoài vào code của mình**

```ts
// API cũ cung cấp callback nhưng bây giờ cần Promise cho code mới

// Adaptee: Thư viện cũ với hàm dùng callback
class OldApi {
  getData(callback: (error: Error | null, data?: string) => void): void {
    // Giả lập xử lý bất đồng bộ
    setTimeout(() => {
      // Gọi callback với dữ liệu
      callback(null, "Kết quả từ OldApi");
    }, 1000);
  }
}

// Target: Ta muốn interface mới trả về Promise<string> khi gọi getData()
interface INewApi {
  getData(): Promise<string>;
}

// Adapter: chuyển đổi OldApi (callback) sang INewApi (promise)
class NewApiAdapter implements INewApi {
  private oldApi: OldApi;
  constructor(oldApi: OldApi) {
    this.oldApi = oldApi;
  }
  getData(): Promise<string> {
    return new Promise((resolve, reject) => {
      this.oldApi.getData((error, data) => {
        if (error) {
          return reject(error);
        }
        resolve(data || "");
      });
    });
  }
}

// Sử dụng Adapter
const oldApi = new OldApi();
const api: INewApi = new NewApiAdapter(oldApi);
api.getData().then(result => {
  console.log(result);  // "Kết quả từ OldApi"
});
```

### Decorator

- là **Structual Pattern**
- **dùng để thêm trách nhiệm hoặc hành vi cho obj tại runtime bằng cách bọc obj đó trong một obj khác gọi là decorator**
- inheritance có mở rộng hành vi tại compile time cho tất cả obj thuộc class còn decorator áp dụng cho từng obj và còn có thể kết hợp nhiều decorator với nhau
- Decorator thường có cùng interface với obj gốc, giữ reference đến obj gốc
- khi decorator implement 1 method, nó có thể gọi method tương ứng (wrap) của obj gốc và bổ sung hành vi trước và sau khi gọi
- các decorator có thể stack lên nhau, và mỗi decorator bọc quanh obj trước đó

```ts
// có obj Server handling req
// muốn có thêm khả năng logging khi handle req nhưng không sửa class Server

// Component gốc
interface IServer {
  handleRequest(req: string): void;
}
class BasicServer implements IServer {
  handleRequest(req: string): void {
    console.log(`Handling request: ${req}`);
    // ... xử lý chính (ví dụ: truy xuất DB, trả về response)
  }
}

// Decorator
class LoggingServerDecorator implements IServer {
  constructor(private wrappedServer: IServer) {}
  handleRequest(req: string): void {
    console.log(`[Log] Received request: ${req}`);
    this.wrappedServer.handleRequest(req);
    console.log(`[Log] Done handling request: ${req}`);
  }
}

// Sử dụng
const server: IServer = new BasicServer();
const loggingServer: IServer = new LoggingServerDecorator(server);

// Gọi phương thức qua decorator
loggingServer.handleRequest("GET /users");
// Console:
// [Log] Received request: GET /users
// Handling request: GET /users
// [Log] Done handling request: GET /users
```

### Proxy

- là **Structural Pattern**
- **dùng để cung cấp 1 obj thay thế cho obj thật để kiểm soát việc access đến obj thật đó**
- Proxy đóng vai trò trung gian đứng trước obj thật:
  - chặn các tương tác tới obj thật, có thể thực hiện thêm một số việc bổ sung (authen, caching, logging...)
  - chỉ tạo obj thật khi cần (dạng virtual proxy, lazy initialization)
  - bảo vệ obj thật khỏi invalid access (protection proxy)
  - cung cấp cách access obj ở xa (remote proxy)
- giống Decorator, Proxy cũng giữ reference đến obj thật và implement cùng interface
- proxy thường tập trung vào việc control access tới obj thật, còn decorator bổ sung tính năng

```ts
// implement proxy cho class API giả định
// class RealAPI sẽ fetch data, proxy sẽ cache result

interface API {
  fetchData(id: string): string;
}

class RealAPI implements API {
  fetchData(id: string): string {
    console.log(`RealAPI: Đang lấy dữ liệu cho ${id}...`);
    // Giả lập hành vi lấy dữ liệu tốn kém
    return `Dữ liệu của ${id}`;
  }
}

class CacheProxy implements API {
  private cache: { [key: string]: string } = {};
  constructor(private realApi: API) {}
  
  fetchData(id: string): string {
    if (this.cache[id]) {
      console.log(`Proxy: Dùng dữ liệu cache cho ${id}`);
      return this.cache[id];
    }
    const result = this.realApi.fetchData(id);
    this.cache[id] = result;
    return result;
  }
}

// Sử dụng
const api = new CacheProxy(new RealAPI());
console.log(api.fetchData("item1"));  // RealAPI được gọi, cache lại
console.log(api.fetchData("item1"));  // Lần này dùng cache
```

## Best Practice

### tách biệt layer

- áp dụng kiến trúc layer, thường là **Controller, Service, Data Access**
  - Controller nhận req, gọi Service, trả res
  - Service handle business logic
  - Data Access handle accessing data

### quản lý config tập trung

- tách config setting (port server, db connection, API key,...) vào module hoặc folder riêng
- không hardcode
- nên sử dụng `dotenv` để quản lý

### logging

- xây dựng cơ chế logging chi tiết để theo dõi hoạt động của app và debug khi cần
- không dùng `console.log` trong prod vì không đủ mạnh và thiếu cấu trúc (có thể dùng Winston hoặc Pino)
- thực hiện **structural logging**: log có cấu trúc JSON hoặc key-value để dễ phân tích
- dùng các mức log như debug, info, warn, error
- mỗi event quan trọng đều nên được log lại cùng context để tracking sau này
- nên có timestamp ở mỗi log

### error handling

- thiết kế cơ chế xử lý lỗi tập trung
- cần tách biệt được lỗi business và lỗi system để có hướng xử lý thích hợp

### testing

- nên có unit test và integration test để cải thiện chất lượng code, giúp phát hiện bug sớm

### CI/CD

- cơ bản gồm các bước:
  - auto testing and analyze code khi có code mới
  - build app
  - deploy nếu 2 bước trên thành công
