# 🟦 Phase 1 — Node.js Core Concepts

## 1️⃣ What is Node.js?
Node.js is a **JavaScript runtime environment** built on the **V8 engine**, used to run JavaScript on the server (outside the browser).  
It provides non-blocking I/O, single-threaded JS + multi-threaded internals, and is ideal for fast, scalable backend systems.

---

## 2️⃣ How Node Works (V8 + Libuv)
Node.js works using two main components:

### ✔️ V8 Engine  
- Executes JavaScript  
- Converts JS → Machine code  
- Extremely fast  

### ✔️ libuv  
- Provides **event loop**  
- Handles **async I/O**  
- Has a **thread pool**  
- Manages timers, file system, DNS, network operations  

Together:  
JS runs on V8 → async tasks go to libuv → event loop picks callbacks → results return to JS.

---

## 3️⃣ Event Loop

When Node.js runs your file, the JavaScript code is first given to the V8 engine, which parses it, compiles it, and converts it into machine code. Once compiled, Node starts executing all synchronous JavaScript on the call stack. The call stack is simply the place where immediate code runs — for example, a console.log("Hi") goes straight to the stack, runs instantly, and is removed. Only one thing runs at a time, because JavaScript is single-threaded.

Whenever your code contains an asynchronous operation such as setTimeout, Promise.then, or fs.readFile, JavaScript doesn’t execute those. Instead, it registers the callback and hands the work to libuv (a C library used internally by Node). Libuv performs the async task in the background while JavaScript continues executing synchronous lines without waiting. This is how Node stays non-blocking.

After the synchronous part of your code finishes for the current turn, Node begins processing anything waiting in the async queues. Node has three important queues, and each one has a different execution priority:

**process.nextTick()** has the highest priority. Anything scheduled using nextTick runs immediately after the current synchronous code completes. This queue runs before every other queue.

**Microtask Queue** is where Promise.then and queueMicrotask callbacks go. Microtasks run right after nextTick, and they run to completion before Node moves on.

**Macrotask Queue** (also called the callback queue) contains things like setTimeout, setInterval, setImmediate, file system events, and network responses. These run only after nextTick and microtasks finish.

Everything — nextTick, microtasks, macrotasks — eventually gets executed on the call stack, because the call stack is the only place JavaScript can actually run code.

To understand this flow, consider your example:

```js
console.log("1");

process.nextTick(() => console.log("nextTick"));

Promise.resolve().then(() => console.log("promise"));

setTimeout(() => console.log("timeout"), 0);

console.log("2");
```

The two console.log statements (1 and 2) run immediately on the call stack because they are synchronous.
After synchronous execution finishes, Node checks the async queues.

First, the nextTick callback runs because it has the highest priority. So "nextTick" goes to the call stack and gets executed immediately.

Next, Node processes the microtask queue, so the promise callback ("promise") is taken and executed on the call stack.

Finally, after microtasks are done, Node moves to the macrotask queue, where the setTimeout callback is waiting. "timeout" is then executed.

### Final Output:
```
1
2
nextTick
promise
timeout
```

### Code Table

| Code | What Happens |
|------|--------------|
| console.log("1") | Goes to Call Stack → print → removed |
| process.nextTick(..) | Callback goes to Next Tick Queue |
| Promise.then(..) | Callback goes to Microtask Queue |
| setTimeout(..) | Timer → after 0ms → callback goes to Macrotask Queue |
| console.log("2") | Goes to Call Stack → print → removed |

Now sync code is done.

---
## 4️⃣ Libuv & Thread Pool


## 🔵 CPU vs IO Tasks (Super Simple)

### **IO Tasks**
Examples:
- Reading/writing files  
- Network requests  
- Database queries  
- Timers  

✔️ Fast for CPU, slow because they wait on external systems  
✔️ Handled by **libuv + OS**, not JavaScript  
✔️ **Do NOT block** the main thread  
✔️ Node continues running other code smoothly  

---

### **CPU Tasks**
Examples:
- bcrypt hashing  
- crypto operations  
- gzip compression  
- image processing  
- heavy JSON parsing  

❗ Heavy computation work  
❗ Executed inside **libuv's thread pool**  
❗ Too many CPU tasks → thread pool busy → slower API responses  

---

## 🔵 Blocking vs Non-Blocking

### **Non-Blocking (Recommended)**
```js
fs.readFile("file.txt", () => {});
```
✔️ Work done by libuv  
✔️ Event loop free  
✔️ Node handles thousands of requests  
✔️ Callback runs later  

---

### **Blocking (Avoid in backend)**
```js
fs.readFileSync("file.txt");
```
❌ Main thread freezes until file completes  
❌ Stops entire event loop  
❌ Only one request runs at a time  
❌ Causes slowdowns under load  

---

## 🔵 UV_THREADPOOL_SIZE (Why This Matters)

- Node’s libuv thread pool has **4 threads by default**
- These threads handle CPU-heavy async tasks like:
  - bcrypt hashing  
  - crypto.pbkdf2  
  - gzip compression  
  - some fs operations  
  - DNS lookups  

---

### **Increase thread pool size if your app does heavy CPU work**
```js
process.env.UV_THREADPOOL_SIZE = 32;
```

Useful when:
- Many users logging in at once (bcrypt heavy)
- File processing service
- Compression/encryption pipelines
- High concurrency with CPU-based tasks

---
