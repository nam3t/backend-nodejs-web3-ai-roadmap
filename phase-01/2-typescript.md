**Table of content:**

- [Generics](#generics)

## Generics

- Generics dùng để viết hàm, class hoặc interface cho phép hoạt động với nhiều kiểu dữ liệu khác nhau
- thay vì dùng một kiểu dữ liệu cụ thể, ta sử dụng một placeholder (thường là `T`) để đại diện cho kiểu dữ liệu sẽ được xác định khi sử dụng

- ví dụ 1:

```js
function identity<T>(arg: T): T {
  return arg;
}

let output1 = identity<string>("Hello");
let output2 = identity<number>(123);
```

  => hàm `identity` trả về giá trị được truyền vào
  => `T` là placeholder cho kiểu dữ liệu, khi gọi hàm sẽ chỉ định kiểu cụ thể

- ví dụ 2:

```js
let output = identity("World"); // TypeScript suy luận T là string
```

  => Typescript có thể tự nhận biết kiểu dữ liệu dựa trên arg

### Generics trong class

- dùng để tạo cấu trúc dữ liệu linh hoạt

```js
class Container<T> {
  private value: T;

  constructor(value: T) {
    this.value = value;
  }

  getValue(): T {
    return this.value;
  }
}

let stringContainer = new Container<string>("Hello");
let numberContainer = new Container<number>(42);
```

### Generics trong interface

```js
interface Pair<T, U> {
  first: T;
  second: U;
}

let pair: Pair<string, number> = { first: "Age", second: 30 };
```

### Generics Constraints

- có thể giới hạn kiểu dữ liệu mà Generics chấp nhận

```js
function getLength<T extends { length: number }>(arg: T): number {
  return arg.length;
}

getLength("Hello"); // OK
getLength([1, 2, 3]); // OK
getLength(123); // Lỗi: number không có thuộc tính length
```

  => `T` giới hạn phải là kiểu có property `length`

### Generics với default value

- chỉ định default value cho Generics

```js
function createArray<T = string>(length: number, value: T): T[] {
  return Array(length).fill(value);
}

let result = createArray(3, "x"); // T mặc định là string
```

## Conditional Types và `infer`

### Conditional Types

- định nghĩa kiểu dữ liệu dựa trên điều kiện, tương tự `? :`

```js
type ConditionalType<T> = T extends U ? X : Y;
```

  => nếu `T` có thể extends `U` thì kết quả là `X` còn không là `Y`

```js
type IsString<T> = T extends string ? true : false;

type A = IsString<'hello'>; // true
type B = IsString<42>;      // false
```

### `infer`

- dùng trong Conditional Types để lấy một phần kiểu dữ liệu và gán nó cho một biến kiểu mới => giúp thao tác với các thành phần cụ thể của kiểu dữ liệu phức tạp
- ví dụ lấy kiểu phần tử của mảng:

```js
type ElementType<T> = T extends (infer U)[] ? U : T;

type A = ElementType<number[]>; // number
type B = ElementType<string>;   // string
```

  => nếu `T` là mảng thì trả về phần tử của mảng, còn không trả về `T`

- ví dụ lấy kiểu trả về của hàm

```js
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = ReturnType<() => string>; // string
type B = ReturnType<(x: number) => boolean>; // boolean
```

- ví dụ lấy kiểu tham số đầu tiên của hàm

```js
type FirstArg<T> = T extends (arg: infer A, ...args: any[]) => any ? A : never;

type A = FirstArg<(name: string, age: number) => void>; // string
```

- ví dụ lấy kiểu dữ liệu từ Promise

```js
type AwaitedType<T> = T extends Promise<infer U> ? U : T;

type A = AwaitedType<Promise<number>>; // number
type B = AwaitedType<string>;          // string
```

## Template Literal Types

- syntax: ``` `${...}` ```
- dùng để tạo các type string mới bằng cách nội suy các type string khác

```js
type World = "world";
type Greeting = `hello ${World}`; // "hello world"
```

  => type `Greeting` có value là `"hello world"`

### Kết hợp với Union Types

- tạo ra tất cả các kết hợp có thể

```js
type Color = "red" | "blue";
type Quantity = "one" | "two";
type SeussFish = `${Quantity | Color} fish`;
// "one fish" | "two fish" | "red fish" | "blue fish"
```

### Sử dụng với `infer` để lấy một phần type của string

```js
type EventName<T extends string> = T extends `on${infer R}Changed` ? R : never;

type Result = EventName<"onUserChanged">; // "User"
```

## Branded Types

- là technique tạo ra các type **nominal** (hữu danh vô thực :v) từ các type cơ bản như `string`, `number`...
  => giúp phân biệt các value cùng kiểu cơ bản nhưng mang ý nghĩa khác nhau trong context app
- vì Typescript sử dụng **structural typing** (2 type cùng cấu trúc sẽ được coi là tương thích)
  => có thể dẫn đến lỗi không mong muốn khi vô tình sử dụng nhầm các biến cùng kiểu nhưng mang ý nghĩa khác nhau

```js
type UserId = number;
type ProductId = number;

function getUser(id: UserId) {
  // ...
}

const productId: ProductId = 123;
getUser(productId); // Không có lỗi, nhưng logic sai
```

- để tạo **Branded Type**, có thể kết hợp một kiểu cơ bản với một thuộc tính đặc biệt chỉ tồn tại trong type của Typescript
- ví dụ `unique symbol` để đảm bảo tính unique của brand

```js
declare const userIdBrand: unique symbol;
type UserId = number & { [userIdBrand]: void };

declare const productIdBrand: unique symbol;
type ProductId = number & { [productIdBrand]: void };
```

  => `UserId` và `ProductId` là 2 kiểu khác nhau dù cùng dựa trên type `number`

- để tạo value thuộc kiểu **Branded**, cần sử dụng ép kiểu (type assertion) hoặc các hàm constructor

```js
function createUserId(id: number): UserId {
  return id as UserId;
}

function createProductId(id: number): ProductId {
  return id as ProductId;
}

const userId = createUserId(1);
const productId = createProductId(2);

// getUser(productId); // Lỗi: Argument of type 'ProductId' is not assignable to parameter of type 'UserId'
```

### Một số trường hợp sử dụng Branded Type

- Phân biệt các loại ID: `UserId`, `ProductId`, `OrderId`... => tránh nhầm lẫn các ID cùng kiểu
- Đơn vị đo lường: `Meters`, `Kilometers`, `Miles`... => tránh lỗi khi thực hiện tính toán
- Data đã xác thực: `EmailAddress`, `PhoneNumber`... => đảm bảo data được check trước khi sử dụng
- Giá trị ràng buộc: `PositiveNumber`, `NonEmptyString`... => đảm bảo constraint của value
