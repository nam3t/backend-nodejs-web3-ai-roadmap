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

### 