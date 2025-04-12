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

## Functional Programming

- Là phương pháp tập trung vào việc sử dụng func như là đơn vị cơ bản để xử lý data, giảm thiểu side effects => tạo ra code dễ maintainance, dễ predict.

### Khái niệm cơ bản

#### 1. Func as First-Class Citizen (Func là công dân hạng nhất)

- Func được xem là obj, có thể gán cho biến, truyền vào func khác đưới dạng args, hoặc trả về từ một func khác.

```js
// Gán func cho biến
const add = (a, b) => a + b;

// Truyền func làm args
function doOperation(a, b, operation) {
  return operation(a, b);
}
console.log(doOperation(3, 5, add)); // 8
```

#### 2. Pure Func

- Một func được coi là **Pure** khi với cùng một input nó luôn trả ra cùng một output và không gây ra side effects (không thay đổi biến ngoài func, không thực hiện I/O...) => giúp code dễ test, dễ predict và an toàn khi thay đổi

```js
// pure func: không thay đổi biến ngoài và luôn cho kết quả nhất định với cùng input
function square(n) {
  return n * n;
}
console.log(square(5)); // 25
```

#### 3. Immutability

- Functional Programming khuyến khích không thay đổi data (immutable data)
- Khi cần thay đổi, tạo ra một bản sao mới thay vì thay đổi biến ban đầu
- Giúp tránh các lỗi unpredictable khi data bị thay đổi ở những chỗ khác

```js
const numbers = [1, 2, 3];
// Thay vì sửa mảng numbers trực tiếp, ta tạo ra một mảng mới bằng cách sử dụng spread operator
const newNumbers = [...numbers, 4];
console.log(newNumbers); // [1, 2, 3, 4]
```

### Higher-Order Func

- Higher-Order Func là các func nhận 1 hoặc nhiều func khác làm args hoặc trả về một func => abstract process & reuse code hiệu quả
- Ví dụ: các func `map`, `filter`, `reduce` đều là Higher-Order Func

```js
const numbers = [1, 2, 3, 4];
const squared = numbers.map(n => n * n); // Áp dụng hàm cho mỗi phần tử của mảng và trả về một mảng mới.
console.log(squared); // [1, 4, 9, 16]

const numbers = [1, 2, 3, 4, 5];
const evenNumbers = numbers.filter(n => n % 2 === 0); // Lọc các phần tử của mảng theo điều kiện cho trước.
console.log(evenNumbers); // [2, 4]

const numbers = [1, 2, 3, 4];
const sum = numbers.reduce((total, n) => total + n, 0); // Gom các giá trị trong mảng thành một giá trị duy nhất.
console.log(sum); // 10
```

### Function Composition và Currying

#### Function Composition

- Là kỹ thuật kết hợp nhiều func với nhau tạo thành 1 func mới => cho phép xử lý data qua nhiều bước theo chuỗi logic

```js
const addOne = x => x + 1;
const double = x => x * 2;

// func composition: tăng 1 sau đó nhân đôi
const addOneThenDouble = x => double(addOne(x));
console.log(addOneThenDouble(5)); // (5 + 1) * 2 = 12
```

- lib support func composition: **Ramda** & **Lodash FP**

#### Currying

- Là quá trình chuyển đổi 1 func có nhiều args thành chuỗi các func, mỗi func có 1 args => giúp tạo ra các func chuyên biệt từ 1 func tổng quát => tăng tính reuse và dễ control logic

```js
// func tổng quát có hai args
const add = (a, b) => a + b;

// func currying: chuyển thành chuỗi hàm
const curriedAdd = a => b => a + b;

const addFive = curriedAdd(5); // Tạo func riêng cộng với 5
console.log(addFive(10)); // 15
```

#### Recursion (đệ quy)

- Trong functional programming, thay vì dùng các vòng lặp như `for`, `while`, ta có thể sử dụng Recursion func, là func gọi lại chính nó
- ==**Note**==: khi sử dụng Recursion func, cần đảm bảo có điều kiện rõ ràng để tránh infinite loop

```js
// tính giai thừa
function factorial(n) {
  if (n === 0) return 1; // Điều kiện dừng
  return n * factorial(n - 1);
}
console.log(factorial(5)); // 120
```

### Ưu và nhược điểm của Functional Programming

#### Ưu

- Dễ test
- Dễ maintainance do giảm độ phức tạp vì được chia thành những func nhỏ, độc lập với nhau
- Abstract hoá: phù hợp pipeline xử lý data

#### Nhược

- Đôi khi khó hiểu
- Performance issue với đệ quy trong một số bài toán

## Modules & ES6+ Features

### Modules

- Modules cho phép chia nhỏ app thành các module độc lập, mỗi module có thể định nghĩa các biến, func, class... public hoặc private => tổ chức code logic hơn & nâng cao khả năng reuse

#### Export

##### 1. named export

```js
// utils.js
export function sum(a, b) {
  return a + b;
}
export const PI = 3.14;
```

- Có thể export nhiều component từ 1 file bằng cách dùng keyword `export` trước mỗi khai báo

##### 2. default export

```js
// math.js
export default function multiply(a, b) {
  return a * b;
}
```

- Một module chỉ có 1 default export, khi import có thể đặt tên tuỳ ý

```js
// main.js
import multiply from './math.js';
console.log(multiply(3, 4));
```

#### Import

##### 1. Import các component sử dụng named export

```js
import { sum, PI } from './utils.js';
console.log(sum(2, 3), PI);
```

##### 2. Import tất cả các export của 1 module, dùng `* as`

```js
import * as utils from './utils.js';
console.log(utils.sum(2, 3), utils.PI);
```

##### 3. Dynamic import: cho phép load module linh hoạt khi runtime => trả về một `Promise`

```js
async function loadModule() {
  const module = await import('./moduleA.js');
  module.someFunction();
}
loadModule();
```

- Dùng để code splitting, chỉ load module khi cần thiết, cải thiện performance.

### Các tính năng ES6 quan trọng

#### Destructuring

- Cho phép tách các value từ array hoặc obj thành các biến riêng biệt => code dễ đọc

```js
// Destructuring obj:
const person = { name: 'Alice', age: 30, city: 'Hanoi' };
const { name, age } = person;
// Destructuring array:
const numbers = [1, 2, 3];
const [first, second] = numbers;
```

#### Spread operator và Rest parameter

- Spread operator `...` dùng để extend một array hoặc obj

```js
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];  // [1, 2, 3, 4, 5]

const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 };  // { a: 1, b: 2, c: 3 }
```

- Rest parameter `...` dùng để gộp các phần tử còn lại của array hoặc obj thành một biến args trong func

```js
function sum(...numbers) {
  return numbers.reduce((acc, curr) => acc + curr, 0);
}
console.log(sum(1, 2, 3, 4)); // 10

const { a, ...others } = { a: 1, b: 2, c: 3 };
console.log(a);     // 1
console.log(others); // { b: 2, c: 3 }
```

#### Arrow Func

```js
const multiply = (a, b) => a * b;
```

- ==Arrow func không có binding riêng cho keyword `this`==

#### Template Literals

- Cho phép tạo multiline cho `string`, giúp dễ insert biến vào `string`

```js
const name = 'Alice';
const greeting = `Hello, ${name}! Welcome to JavaScript ES6.`;
// Hỗ trợ multiline string:
const multiline = `Dòng 1
Dòng 2
Dòng 3`;
```

#### Optional Chaining và Nullish Coalescing operator

- Optional Chaining `?.` dùng để kiểm tra nếu một obj có tồn tại trước khi access vào property của nó => để không xảy ra lỗi runtime

```js
const user = { profile: { email: 'alice@example.com' } };
console.log(user.profile?.email);  // 'alice@example.com'
console.log(user.address?.city);   // undefined
```

- Nullish Coalescing operator `??` trả về giá trị bên phải khi giá trị bên trái là `null` hoặc `undefined`

```js
const value = null;
const result = value ?? 'Default value';
console.log(result); // 'Default value'
```

#### Các tính năng khác

- `let` và `const` cung cấp cách khai báo biến với block scope, thay thế `var` với hạn chế về hoisting

```js
let count = 0;
const PI = 3.14;
```

- `class` cho phép định nghĩa các class gần giống với các ngôn ngữ OOP truyền thống

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    console.log(`${this.name} makes a sound.`);
  }
}
const dog = new Animal('Buddy');
dog.speak();
```

- `Promise` hỗ trợ xử lý asynchronous tốt hơn so với callback

```js
const asyncTask = new Promise((resolve, reject) => {
  setTimeout(() => resolve('Done!'), 1000);
});
asyncTask.then(result => console.log(result));  // Hiển thị "Done!" sau 1 giây
```

### Tại sao nên sử dụng ES6+ và Module

- **Tổ chức code tốt hơn**: Module giúp chia nhỏ code thành các phần riêng biệt => dễ quản lý và maintain
- **Dễ Extend**: khi dự án phát triển lớn hơn, module giúp dễ dàng update, maintain mà không ảnh hưởng đến các phần khác
- **Ngắn gọn**: với syntax mới từ ES6, code trở nên ngắn gọn, rõ ràng hơn
- **Hỗ trợ asynchronous hiệu quả**: với sự xuất hiện của `Promise` và `async/await`, xử lý asynchronous trở nên dễ dàng và hiệu quả hơn
- **Bảo mật và tránh xung đột**: Module giúp tránh việc conflict do trùng lặp tên

