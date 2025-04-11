# Advanced Javascript

## Scope, Closure & Hoisting

### Scope

- Scope xác định phạm vi mà một variable có thể được truy cập.
- Có 3 loại scope: **Global scope**, **Function scope** và **Block scope**

#### Global scope

- Một biến khi được khai báo không nằm trong một block nào sẽ có **Global scope**
- Có thể được truy cập từ bất kỳ đâu

```js
var globalVar = 'global variable';

function show() {
  console.log(globalVar); // truy cập được biến global
}

show();
console.log(globalVar);
```

#### Function scope

- Các biến khi khai báo trong một func chỉ có thể truy cập ở trong func đó hoặc các func con của func đó (nested func)

```js
function testFunctionScope() {
  var localVar = 'local var';
  console.log(localVar); // Hợp lệ
}

testFunctionScope();
// console.log(localVar);  // Sẽ báo lỗi vì localVar không thể truy cập bên ngoài hàm.
```

#### Block scope

- Từ `ES6`, Javascript hỗ trợ block scope (nằm trong một block `{ }`) với 2 keywords `let` và `const`.

```js
if (true) {
  let blockVar = 'only exist in this block';
  console.log(blockVar);  // Hợp lệ
}
// console.log(blockVar); // Sẽ báo lỗi vì blockVar không tồn tại bên ngoài khối if.
```

==*Note*: Cách khai báo với keyword `var` sẽ không có **block scope** mà chỉ có **function scope**==

### Closure

- Closure là khi một func ghi nhớ được một biến của phạm vi bên ngoài nó (`lexical enviorment`) ngay cả khi func đó đã chạy xong.
- Về cơ bản, **Closure** cho phép func con ghi nhớ nơi nó được khai báo, qua đó có thể truy cập được biến của func cha kể cả khi func cha đã kết thúc.

```js
function createCounter() {
  let count = 0; // biến count thuộc phạm vi của hàm createCounter

  return function() { // Func này trở thành closure
    count++;       // Nó có thể truy cập biến count
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // In ra 1
console.log(counter()); // In ra 2
```

- Khi khởi tạo fun `createCounter()`, biến `count` được giữ lại trong bộ nhớ bởi closure
- Func closure trả về có thể truy cập và thay đổi biến `count` dù func `createCounter()` đã kết thúc

#### Note

- Mỗi khi tạo ra Closure, một vùng nhớ riêng biệt giữ lại các biến của môi trường sẽ được tạo.
- Ứng dụng cho việc **bảo vệ những dữ liệu riêng tư** và **tạo ra `stateful func`**
- Closure hữu ích trong các module Javascript để giới hạn scope của biến global.

### Hoisting

- Hoisting là việc khai báo biến (với keyword `var`) và func được đẩy lên đầu scope của chúng khi code được chạy.
- Chỉ có khai báo được đẩy lên (hoisted), không bao gồm giá trị
  
#### Hoisting biến với `var`

```js
console.log(a); // In ra undefined, chứ không báo lỗi
var a = 10;
console.log(a); // In ra 10
```

- Khi code chạy, khai báo `var a` được hoisted lên đầu, nhưng gán giá trị `10` vẫn xảy ra tại vị trí ban đầu. Do đó, trước khi gán, `a` đã được khai báo nhưng có giá trị mặc định là `undefined`.

#### Hoisting func 

```js
sayHello(); // In ra "Hello World!"

function sayHello() {
  console.log("Hello World!");
}
```

- Các khai báo func được hoisted hoàn toàn (bao gồm cả phần thân hàm), vì vậy có thể gọi func trước khi func được định nghĩa.

#### Hoisting với `let`/`const`

```js
// console.log(b);  // Sẽ báo lỗi ReferenceError
let b = 20;
console.log(b);    // In ra 20
```

- Các biến được khai báo bằng `let` và `const` cũng được hoisted nhưng không được khởi tạo. Khi truy cập trước khi khai báo, sẽ xảy ra lỗi `ReferenceError` do vùng gọi còn được gọi là **Temporal Dead Zone**.

## Asynchronous Programming

### Tại sao cần Asynchronous Programming (Lập trình bất đồng bộ)?

- Trước hết, hãy nói về **Synchronous Programming** (Lập trình đồng bộ), có nghĩa là các câu lệnh được thực hiện theo thứ tự từ trên xuống dưới
- Ví dụ: nếu có 3 tasks A, B, C thì ta sẽ thực hiện A xong B rồi đến C
- Nhưng thực tế, một số task như query db, call API, hay read/write file thường sẽ mất thời gian lớn
- Nếu chờ một task hoàn thành thì toàn bộ app sẽ bị block và không thể phản hồi các event khác
- Khi đó, **Asynchronous Programming** cho phép thực hiện các task nặng mà không block main thread
- Trong khi chờ các task nặng hoàn thành, chương trình vẫn có thể làm việc khác
=> Cải thiện performance và sử dụng resource một cách hợp lý.

### Ví dụ về Async trong Javascript

#### Callback

- Callback là cách xử lý async truyền thống của Javascript
- Ví dụ `setTimeout()` sẽ khai báo một callback func để trả về sau một khoảng thời gian
- Nhược điểm của callback là khi có nhiều func lồng nhau, nó sẽ dễ gây ra **callback hell** làm code khó đọc và khó maintainance.

### Promise

- Promise cho phép xử lý task async qua các method như `.then()`, `.catch()` => giúp việc kết nối các task trở nên rõ ràng hơn
- Khi một Promise được **resolved** hoặc **rejected**, ta có thể xử lý kết quả mà không cần lồng callback quá sâu

### Async/Await

- Đây là cách triển khai sử dụng Promise nhưng dễ đọc hơn, cho phép viết code asynchronous theo kiểu synchronous

```js
async function fetchData() {
  try {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    console.log(data);
  } catch (error) {
    console.error('Lỗi:', error);
  }
}

fetchData();
```

- Ở đây, `await` khiến Javascript chờ Promise fetch API hoàn thành mà không block toàn bộ app.

## Prototypes & Inheritance

### Prototype trong Javascript

- Mỗi một object trong Javascript đều có 1 thuộc tính ẩn gọi là **prototype**, đây là 1 obj mà obj hiện tại inherit các properties và method
- Khi access vào 1 property của 1 obj mà không tìm thấy tại obj đó, thì Javascript sẽ tự động tìm lên đến **prototype** của obj đó để tìm. Quá trình này tiếp tục cho đến khi tìm đến obj gốc (thường là `Object.prototype`)
- **Prototype** giúp **reuse code** và **dynamic inherit**
- **Reuse code**: thay vì viết cùng 1 func ở mỗi obj, ta viết 1 lần ở trên **prototype** để tất cả các obj cùng kiểu sẽ dùng được => giúp tiết kiệm memory
- **Dynamic inherit**: cho phép các obj có thể inherit property và method được định nghĩa trên **prototype** => giúp dễ dàng mở rộng tính năng

### Cách inherit dựa trên prototype (2 cách phổ biến)

#### 1. Constructor Func

- Đây là cách dùng phổ biến trước khi có `ES6`, dùng **Constructor Func** rồi gán method cho prototype

```js
// Constructor Func cho obj Person
function Person(name) {
  this.name = name;
}

// Thêm method sayHello vào prototype của Person
Person.prototype.sayHello = function() {
  console.log("Hello, my name is " + this.name);
};

const alice = new Person("Alice");
alice.sayHello(); // Kết quả: Hello, my name is Alice
```

- Khi gọi `new Person("Alice")`, một obj mới được tạo ra và property `name` được gán giá trị `"Alice"`.
- Method `sayHello` được định nghĩa trên `Person.prototype` nên mọi đối tượng tạo ra từ `Person` đều có thể dùng được mà không cần lưu nhiều bản sao của hàm.

#### 2. ES6 Class

- Từ `ES6`, Javascript giới thiệu keyword `class` giúp code dễ đọc, nhưng bản chất vẫn là sử dụng `prototype`

```js
// Khai báo class Person
class Person {
  constructor(name) {
    this.name = name;
  }
  
  sayHello() {
    console.log(`Hello, my name is ${this.name}`);
  }
}

const bob = new Person("Bob");
bob.sayHello(); // Kết quả: Hello, my name is Bob
```

- **Inheritance với ES6 Class**: có thể dễ dàng mở rộng một class để tạo ra class con bằng cách sử dụng từ khóa `extends`.

```js
class Employee extends Person {
  constructor(name, position) {
    super(name); // Gọi constructor của lớp Person
    this.position = position;
  }
  
  showInfo() {
    console.log(`Name: ${this.name}, Position: ${this.position}`);
  }
}

const charlie = new Employee("Charlie", "Developer");
charlie.sayHello();  // inherit method từ Person
charlie.showInfo();  // method riêng của Employee
```

- `extends` cho biết `Employee` inherit từ `Person`, nên tất cả method định nghĩa trong `Person` (như `sayHello`) đều có thể sử dụng trong `Employee`.
- `super(name)` gọi `constructor` của class cha (`Person`) để thiết lập những property cơ bản trước khi mở rộng thêm các property mới (ở đây là `position`).
