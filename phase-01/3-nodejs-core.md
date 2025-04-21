**Table of content:**

- [Event Loop](#event-loop)
  - [Cách hoạt động](#cách-hoạt-động)
  - [Tránh block Event Loop](#tránh-block-event-loop)
    - [Cách phòng tránh việc block Event Loop](#cách-phòng-tránh-việc-block-event-loop)
- [Cluster \& Worker Thread](#cluster--worker-thread)
  - [Cluster](#cluster)
  - [Worker Thread](#worker-thread)
  - [So sánh](#so-sánh)
- [module `fs`](#module-fs)
  - [Các method chính](#các-method-chính)
    - [read file](#read-file)
    - [write file](#write-file)
  - [Note](#note)
- [module `http`](#module-http)
  - [tạo server HTTP cơ bản](#tạo-server-http-cơ-bản)
  - [xử lý HTTP method và URL](#xử-lý-http-method-và-url)
  - [query string](#query-string)
  - [xử lý POST method](#xử-lý-post-method)
- [module `events`](#module-events)
  - [cách sử dụng](#cách-sử-dụng)
    - [tạo `EventEmitter` và listen event](#tạo-eventemitter-và-listen-event)
    - [sử dụng `once` để listen một lần duy nhất](#sử-dụng-once-để-listen-một-lần-duy-nhất)
    - [gỡ bỏ listener với `removeListener` hoặc `off`](#gỡ-bỏ-listener-với-removelistener-hoặc-off)
  - [các event đặc biệt trong `EventEmitter`](#các-event-đặc-biệt-trong-eventemitter)
    - [`newListener`](#newlistener)
    - [`removeListener`](#removelistener)
    - [`error`](#error)
  - [use case của `EventEmitter`](#use-case-của-eventemitter)
    - [quản lý async process](#quản-lý-async-process)
    - [giao tiếp giữa các module](#giao-tiếp-giữa-các-module)
- [module `stream`](#module-stream)
  - [các loại Stream trong NodeJS](#các-loại-stream-trong-nodejs)
    - [Readable](#readable)
    - [Writable](#writable)
    - [Duplex](#duplex)
    - [Transform](#transform)
  - [handle lỗi và backpressure](#handle-lỗi-và-backpressure)
    - [error](#error-1)
    - [backpressure](#backpressure)


## Event Loop

- **Event Loop** là cơ chế core trong Node cho phép thực hiện non-blocking I/O operations bằng cách offloading các task cho hệ điều hành hoặc các thread khác và tiếp tục thực hiện các task khác trong khi chờ phản hồi.
- NodeJS hoạt động đơn luồng (single-threaded), nhưng với Event Loop nó có thể xử lý đồng thời hàng nghìn kết nối một cách hiệu quả

### Cách hoạt động

- khi NodeJS chạy, nó sẽ thực hiện các bước sau:
  1. **Chạy code đồng bộ**: code được chạy từ trên xuống
  2. **Gửi async task**: các hàm như `setTimeout`, `fs.readFile` được gửi đến API của NodeJS để xử lý
  3. **Event Loop check Call Stack**: nếu Call Stack trống, Event Loop sẽ lấy callback từ queue và đẩy vào Call Stack

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

- Event Loop bao gồm các phase sau:
  - **Timer**: xử lý callback từ `setTimeout` và `setInterval`
  - **Pengding Callback**: xử lý các callback bị trì hoãn từ giai đoạn trước
  - **Idle, Prepare**: dành cho nội bộ NodeJS
  - **Poll**: lấy các event mới, thực hiện I/O
  - **Check**: xử lý callback từ `setImmediate`
  - **Close Callback**: xử lý event liên quan đến đóng kết nối như `socket.on('close')`

- ngoài ra còn có **Microtask Queue** bao gồm `process.nextTick()` và các Promise, được chạy với độ ưu tiên cao nhất, trước mỗi giai đoạn

- ví dụ:

```js
const fs = require('fs');

console.log('1. Start');

setTimeout(() => {
  console.log('2. setTimeout');
}, 0);

setImmediate(() => {
  console.log('3. setImmediate');
});

fs.readFile(__filename, () => {
  console.log('4. fs.readFile callback');

  setImmediate(() => {
    console.log('5. setImmediate inside fs.readFile');
  });

  process.nextTick(() => {
    console.log('6. process.nextTick inside fs.readFile');
  });

  Promise.resolve().then(() => {
    console.log('7. Promise inside fs.readFile');
  });
});

process.nextTick(() => {
  console.log('8. process.nextTick');
});

Promise.resolve().then(() => {
  console.log('9. Promise');
});

console.log('10. End');
```

1. code đồng bộ `console.log('1. Start')` và `console.log('10. End')` chạy ngay
2. `process.nextTick()` và Promise:
    - `process.nextTick(() => console.log('8. process.nextTick'))` được thực thi ngay sau code đồng bộ.
    - `Promise.resolve().then(() => console.log('9. Promise'))` được thực thi sau `process.nextTick()`.​
3. `setTimeout()` và `setImmediate()`:
    - `setTimeout(() => console.log('2. setTimeout'))` được lên lịch trong **Timers** phase.
    - `setImmediate(() => console.log('3. setImmediate'))` được lên lịch trong **Check** phase.​
4. `fs.readFile()`:
    - Là một I/O operation, callback của nó được thực thi trong **Poll** phase.
    - Bên trong callback của `fs.readFile()`, các hàm `setImmediate`, `process.nextTick`, và Promise được lên lịch tương ứng trong các phase và queue của chúng.​

```js
// output
1. Start
10. End
8. process.nextTick
9. Promise
2. setTimeout
3. setImmediate
4. fs.readFile callback
6. process.nextTick inside fs.readFile
7. Promise inside fs.readFile
5. setImmediate inside fs.readFile
```

==Thứ tự giữa `setTimeout()` và `setImmediate()` có thể thay đổi tùy thuộc vào môi trường thực thi. Tại vì:==

1. **Timing thực tế không tuyệt đối**
    - `setTimeout(..., 0)` không thực sự chạy ngay lập tức sau `0ms` – nó được lên lịch để không sớm hơn `0ms`.
    - Tùy hệ điều hành và load hệ thống, thời gian thực thi có thể bị delay vài `ms`, khiến thứ tự thay đổi.

2. **Thread pool của libuv**
    - Node.js dùng `libuv` để xử lý async I/O (ví dụ `fs.readFile()`).
    - `libuv` có thread pool, và thời gian hoàn thành tác vụ phụ thuộc vào:
      - Độ lớn file
      - Hệ thống file
      - Load CPU / Disk
    - Vì vậy thời điểm callback của I/O (poll phase) có thể sớm hoặc muộn hơn kỳ vọng.

3. **Cuộc đua giữa poll vs check phase**
    - `setImmediate()` và I/O callback (ví dụ `fs.readFile`) có thể chạy trước hoặc sau nhau, tùy vào khi nào poll kết thúc.
    - Nếu poll phase hoàn thành trước khi có I/O callback sẵn sàng, check phase (tức setImmediate) có thể chạy trước.
    - Nếu có I/O callback sẵn sàng ngay, nó có thể chạy trước setImmediate.

```js
fs.readFile(__filename, () => {
  console.log('fs.readFile callback');
});
setImmediate(() => {
  console.log('setImmediate');
});
```

- Thứ tự output phụ thuộc vào timing chính xác trong vòng lặp hiện tại của event loop.

4. **Phiên bản Node.js**
    - Mỗi phiên bản Node.js có thể implement Event Loop theo cách hơi khác nhau (đặc biệt trong các bản update lớn).
    - Thứ tự xử lý nextTick / Promise có thể tối ưu lại sau các bản vá.

5. **Load hệ thống, môi trường ảo hóa, container**
    - Khi chạy trong môi trường Docker, máy ảo (VM), hoặc CPU đang bận, các tác vụ I/O và event scheduling có thể lệch timing.

### Tránh block Event Loop

- vì là single-threaded nên khi gặp những task nặng, Event Loop có thể bị block, gây ra giảm performance
- ví dụ vòng lặp vô hạn

```js
let flag = false;

setTimeout(() => {
  flag = true;
  console.log('Timeout executed');
}, 1000);

while (!flag) {
  // Vòng lặp này chặn Event Loop
}
```

=> vòng lặp `while` chạy liên tục cho đến khi `flag` trở thành `true`, nhưng `setTimeout` không thể thực thi callback của nó vì Event Loop đang bị chặn bởi vòng lặp => treo app

- ví dụ sử dụng hàm đồng bộ nặng

```js
const fs = require('fs');

setTimeout(() => {
  console.log('Timeout executed');
}, 100);

const data = fs.readFileSync('/path/to/large/file.txt');
console.log('File read complete');
```

=> `fs.readFileSync` đọc file đồng bộ nên sẽ chặn Event Loop cho đến khi file đọc xong => delay `setTimeout` => giảm performance

- ví dụ xử lý JSON nặng

```js
let obj = { a: 1 };
for (let i = 0; i < 20; i++) {
  obj = { obj1: obj, obj2: obj }; // Tăng kích thước đối tượng theo cấp số nhân
}

const jsonString = JSON.stringify(obj); // Có thể mất nhiều thời gian
console.log('JSON.stringify complete');
```

=> `JSON.stringify` xử lý obj nặng có thể mất nhiều thời gian, chặn Event Loop và giảm performance

#### Cách phòng tránh việc block Event Loop

- sử dụng hàm async thay vì hàm sync (`fs.readFile` thay vì `fs.readFileSync`)
- chia nhỏ task nặng và xử lý async
- sử dụng `setImmediate` hoặc `process.nextTick` để delay các task nặng, để Event Loop xử lý các task khác trước
- sử dụng **worker threads** đễ xử lý task nặng về CPU trong các thread riêng, tránh chặn thread chính.

## Cluster & Worker Thread

### Cluster

- mặc định, NodeJS chạy single thread và chỉ sử dụng 1 nhân CPU
- module `cluster` giúp tạo ra nhiều **worker process**, mỗi một process chạy trên một nhân CPU riêng biệt
  => tăng performance xử lý concurrent connection nhờ phân phối cho các worker process.
  => tận dụng tài nguyên CPU
  => tăng tính chịu tải: nếu một worker lỗi, master process có thể tạo lại worker mới để tiếp tục

```js
const cluster = require('cluster');
const os = require('os');
const express = require('express');

const numCPUs = os.cpus().length;

if (cluster.isMaster) {
  console.log(`Master process PID: ${process.pid}`);

  // Tạo worker process
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // listen event khi worker bị exit
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker PID: ${worker.process.pid} died. Restarting...`);
    cluster.fork();
  });
} else {
  const app = express();

  app.get('/', (req, res) => {
    res.send(`Hello from worker PID: ${process.pid}`);
  });

  app.listen(3000, () => {
    console.log(`Worker PID: ${process.pid} is listening on port 3000`);
  });
}
```

- master process tạo ra số worker process tương ứng với số nhân CPU
- Mỗi worker chạy một instance của Express và listen trên cùng một port

### Worker Thread

- module `worker_threads` giúp tạo ra các thread để chạy song song các task nặng về CPU mà không block event loop chính
- mỗi worker thread sẽ có 1 event loop riêng và có thể share memory với thread chính qua `SharedArrayBuffer`
- usecase của worker thread:
  - xử lý task tính toán phức tạp: mã hoá, giải mã, xử lý ảnh/video
  - phân tích big data, các thuật toán nặng
  - cần share memory giữa các thread để tăng performance

```js
// main.js
const { Worker } = require('worker_threads');

function runWorker() {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js');

    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', code => {
      if (code !== 0)
        reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

runWorker().then(result => {
  console.log(`Result from worker: ${result}`);
}).catch(err => {
  console.error(err);
});
```

```js
// worker.js
const { parentPort } = require('worker_threads');

// task tính toán nặng
function heavyComputation() {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  }
  return sum;
}

parentPort.postMessage(heavyComputation());
```

- trong ví dụ trên, task nặng được chạy ở worker thread, giúp thread chính không bị block và có thể tiếp tục xử lý các request khác

### So sánh

|  | Cluster | Worker Thread |
| ----------- | ----------- |----------- |
| mục đích chính | tận dụng nhân CPU để xử lý request I/O | xử lý task nặng về CPU |
| cơ chế | tạo nhiều process con | tạo nhiều thread trong cùng 1 process |
| share memory | không | có (qua `SharedArrayBuffer`) |
| giao tiếp | thông qua IPC (Inter-Process Communication) | thông qua `postMessage` và `parentPort` |
| app | web API | task nặng tính toán, big data process |

## module `fs`

- module `fs` (file system) giúp thao tác với các file của server:
  - read/write file
  - quản lý thư mục: create, delete, read directory...
  - theo dõi sự thay đổi của file (`fs.watch`)
  - xử lý I/O async
- module `fs` có 2 kiểu API: sync và async

### Các method chính

#### read file

- `readFile`: đọc async toàn bộ file
- `readFileSync`: đọc sync toàn bộ file

```js
// readFile
const fs = require('fs');

fs.readFile('example.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error reading file:', err);
    return;
  }
  console.log('File content:', data);
});
```

- `fs.readFile` nhận input là **tên file**, **định dạng** (ở đây `utf8` để đảm bảo data đọc về là chuỗi) và **callback**

```js
// readFileSync
const fs = require('fs');

try {
  const data = fs.readFileSync('example.txt', 'utf8');
  console.log('File content:', data);
} catch (err) {
  console.error('Error reading file:', err);
}
```

- `fs.readFileSync` đọc file đồng bộ nên app sẽ chờ đến khi quá trình đọc file hoàn tất

#### write file

- `writeFile`: ghi file async
- `writeFileSync`: ghi file sync

```js
// writeFile
const fs = require('fs');

const content = 'Dữ liệu cần ghi vào file!';

fs.writeFile('output.txt', content, 'utf8', err => {
  if (err) {
    console.error('Error writing file:', err);
    return;
  }
  console.log('File was written successfully!');
});
```

- `fs.writeFile` nhận input là **tên file**, **nội dung**, **encoding** và **callback**
- nếu thành công, file `output.txt` sẽ được tạo ra (hoặc ghi đè lên file cũ)

```js
// writeFileSync
const fs = require('fs');

const content = 'Ghi dữ liệu một cách đồng bộ';
try {
  fs.writeFileSync('output_sync.txt', content, 'utf8');
  console.log('File written successfully!');
} catch (err) {
  console.error('Error writing file:', err);
}
```

### Note

- nên sử dụng version async
- luôn check và handle error
- chú ý tới encoding khi read/write file để tránh các lỗi về format

## module `http`

- module `http` giúp tạo server HTTP và xử lý các request http từ client
- hỗ trợ xây dựng webapp, API RESTful...

### tạo server HTTP cơ bản

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World!');
});

server.listen(3000, () => {
  console.log('Server running at http://localhost:3000/');
});
```

- `http.createServer()` tạo một server HTTP mới
- hàm callback nhận 2 arg: `req` (request từ client) và `res` (response từ server)
- `res.writeHead(200, {'Content-Type': 'text/plain'})` setup status code HTTP và title response
- `res.end('Hello World!')` gửi nội dung response và kết thúc
- `server.listen(3000)` làm server listen trên port 3000

### xử lý HTTP method và URL

```js
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/') {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end('Trang chủ');
  } else if (req.method === 'GET' && req.url === '/about') {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end('Giới thiệu');
  } else {
    res.writeHead(404, {'Content-Type': 'text/plain'});
    res.end('Không tìm thấy');
  }
});

server.listen(3000, () => {
  console.log('Server running at http://localhost:3000/');
});
```

### query string

- sử dụng module `url`

```js
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url, true);
  const query = parsedUrl.query;

  res.writeHead(200, {'Content-Type': 'application/json'});
  res.end(JSON.stringify(query));
});

server.listen(3000, () => {
  console.log('Server running at http://localhost:3000/');
});
```

- khi truy cập link: `http://localhost:3000/?name=santi&age=17`, server sẽ trả về: `{"name":"santi","age":"17"}`

### xử lý POST method

- cần listen event `data` và `end` trên obj `req`

```js
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.method === 'POST') {
    let body = '';

    req.on('data', chunk => {
      body += chunk.toString();
    });

    req.on('end', () => {
      res.writeHead(200, {'Content-Type': 'text/plain'});
      res.end(`Dữ liệu nhận được: ${body}`);
    });
  } else {
    res.writeHead(405, {'Content-Type': 'text/plain'});
    res.end('Chỉ hỗ trợ phương thức POST');
  }
});

server.listen(3000, () => {
  console.log('Server running at http://localhost:3000/');
});
```

## module `events`

- module `events` dùng class core `EventEmitter` giúp các obj tạo và listen các event tuỳ chỉnh
  => tạo app theo model event-driven, giúp quản lý task async hiệu quả

### cách sử dụng

#### tạo `EventEmitter` và listen event

```js
const EventEmitter = require('events');
const eventEmitter = new EventEmitter();

// Đăng ký một listener cho event 'greet'
eventEmitter.on('greet', (name) => {
  console.log(`Hí, ${name}!`);
});

// emit event 'greet'
eventEmitter.emit('greet', 'santi');
```

- `on(eventName, listener)`: đăng ký listener cho event `eventName`
- `emit(eventName, [...args])`: emit event `eventName` kèm theo các args

#### sử dụng `once` để listen một lần duy nhất

```js
eventEmitter.once('onlyOnce', () => {
  console.log('Sự kiện này chỉ được lắng nghe một lần.');
});

eventEmitter.emit('onlyOnce'); // Được xử lý
eventEmitter.emit('onlyOnce'); // Không được xử lý
```

#### gỡ bỏ listener với `removeListener` hoặc `off`

```js
function response() {
  console.log('Listener đã được gọi.');
}

eventEmitter.on('removeTest', response);
eventEmitter.removeListener('removeTest', response);
// Hoặc sử dụng off (từ Node.js v10 trở lên)
eventEmitter.off('removeTest', response);

eventEmitter.emit('removeTest'); // Không có output
```

### các event đặc biệt trong `EventEmitter`

#### `newListener`

- được emit khi một listener mới được thêm

```js
eventEmitter.on('newListener', (event, listener) => {
  console.log(`Listener mới được thêm cho sự kiện: ${event}`);
});
```

#### `removeListener`

- được emit khi một listener bị remove

```js
eventEmitter.on('removeListener', (event, listener) => {
  console.log(`Listener đã bị gỡ bỏ khỏi sự kiện: ${event}`);
});
```

#### `error`

- nếu một event `error` được emit mà không có listener nào thì NodeJS sẽ throw ra lỗi và exit

```js
eventEmitter.on('error', (err) => {
  console.error('Đã xảy ra lỗi:', err);
});

eventEmitter.emit('error', new Error('Lỗi mẫu'));
```

### use case của `EventEmitter`

#### quản lý async process

```js
class Task extends EventEmitter {
  start() {
    this.emit('start');
    setTimeout(() => {
      this.emit('progress', 50);
      setTimeout(() => {
        this.emit('progress', 100);
        this.emit('complete');
      }, 1000);
    }, 1000);
  }
}

const task = new Task();

task.on('start', () => console.log('Bắt đầu tiến trình...'));
task.on('progress', (percent) => console.log(`Tiến độ: ${percent}%`));
task.on('complete', () => console.log('Tiến trình hoàn tất.'));

task.start();
```

#### giao tiếp giữa các module

- để tách biệt logic và tăng tính module, dùng `EventEmitter` để giao tiếp

## module `stream`

- `Stream` là abstract interface dùng để streaming data
- thay vì read/write toàn bộ data cùng một lúc, `Stream` xử lý data theo từng phần nhỏ (chunk) giúp tiết kiệm memory và tăng performance
  => không cần load toàn bộ data vào memory
  => giảm độ trễ khi xử lý data
  => các module core như `fs`, `http`, `net` đều hỗ trợ `Stream`

### các loại Stream trong NodeJS

#### Readable

- read data từ source (file, HTTP request...)

```js
// đọc file
const fs = require('fs');

const readableStream = fs.createReadStream('input.txt', { encoding: 'utf8' });

readableStream.on('data', chunk => {
  console.log('Received chunk:', chunk);
});

readableStream.on('end', () => {
  console.log('Finished reading file.');
});
```

#### Writable

- write data đến destination (file, HTTP response)

```js
// ghi file
const fs = require('fs');

const writableStream = fs.createWriteStream('output.txt');

writableStream.write('Hello, ');
writableStream.write('World!');
writableStream.end();

writableStream.on('finish', () => {
  console.log('Finished writing to file.');
});
```

#### Duplex

- vừa read vừa write (socket)

```js
// sao chép file
const fs = require('fs');

const readableStream = fs.createReadStream('input.txt');
const writableStream = fs.createWriteStream('output.txt');

readableStream.pipe(writableStream);
```

#### Transform

- Duplex stream có thể biến đổi data khi read/write (nén, mã hoá)

```js
// chuyển thành chữ hoa ghi vào file
const fs = require('fs');
const { Transform } = require('stream');

const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

const readableStream = fs.createReadStream('input.txt');
const writableStream = fs.createWriteStream('output.txt');

readableStream.pipe(upperCaseTransform).pipe(writableStream);
```

### handle lỗi và backpressure

#### error

- luôn listen event `error` để xử lý

```js
readableStream.on('error', err => {
  console.error('Error reading file:', err);
});

writableStream.on('error', err => {
  console.error('Error writing file:', err);
});
```

#### backpressure

- khi tốc độ ghi chậm hơn đọc, cần kiểm soát backpressure

```js
const fs = require('fs');

const readableStream = fs.createReadStream('input.txt');
const writableStream = fs.createWriteStream('output.txt');

readableStream.on('data', chunk => {
  const canContinue = writableStream.write(chunk);
  if (!canContinue) {
    readableStream.pause();
  }
});

writableStream.on('drain', () => {
  readableStream.resume();
});
```
