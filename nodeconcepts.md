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

## 5️⃣ Streams & Buffers

When Node.js works with files, network responses, images, videos, or any large amount of data, it cannot load everything into memory at once. If you use `fs.readFile()` on a 5GB file, Node tries to load all 5GB into RAM. This can easily crash your server when many users request large files at the same time. To prevent memory overload, Node provides **Buffers** and **Streams** — tools that let you handle data piece by piece instead of all at once.

A **Buffer** is a small temporary storage area for raw binary data (bytes). Streams use Buffers internally, which means you don’t manually create Buffers when reading files — Node automatically fills Buffers behind the scenes. When you call `fs.createReadStream("sm.txt")`, Node reads raw bytes into a Buffer first, then converts those bytes into a string only if you specify an encoding (like utf8). If you don’t specify encoding, the `"data"` event gives you a Buffer object directly. These Buffers are stored in Node’s internal memory (in the JS heap area), not on disk; they are cleared as soon as the chunk is processed.

A **Stream** lets data flow chunk by chunk instead of waiting for the entire data to load. This allows Node to start processing immediately, improving speed and memory efficiency. Streams are everywhere in Node: reading files, writing files, HTTP requests, HTTP responses, zipping files, uploading, downloading, and sockets.

A **Readable Stream** is something you read from, such as `fs.createReadStream("bigfile.txt")`.  
A **Writable Stream** is something you write to, like `fs.createWriteStream("copy.txt")`.  
Node processes each chunk as it arrives, which keeps memory usage extremely low.

When you call `stream.on("data", callback)`, you’re telling Node:  
**“Whenever a new chunk arrives, give it to me.”**  
- `"data"` fires every time a chunk is ready.  
- `"end"` fires when there is no more data to read.  
If the file is empty, `"data"` never fires, but `"end"` fires immediately.

Node does not load entire files into memory. For a 1GB file, Node splits it into chunks (default ~64KB each). Only one chunk exists in memory at a time. Node delivers that chunk, clears it, and loads the next one. This keeps memory safe for large files.

## 1. What does `readable.pipe(writable)` actually mean?

It means: **“Take the chunks coming from this readable stream and send them directly into this writable stream.”**

Readable = **source**  
Writable = **destination**  
Pipe() = **connection/tunnel between them**

```js
const readable = fs.createReadStream("bigfile.txt");
const writable = fs.createWriteStream("copy.txt");

readable.pipe(writable);
```

This means:

- readable gives chunks →  
- pipe() forwards them →  
- writable writes them to disk  

## 2. Why do we use `pipe()`?

Because manually writing this:

```js
readable.on("data", chunk => writable.write(chunk));
readable.on("end", () => writable.end());
```

`pipe()` does this automatically **plus**:

- handles **backpressure**  
- pauses readable if writable is slow  
- resumes readable when writable is ready  
- avoids memory overflow  

So instead of managing all that manually, you write just:

```js
readable.pipe(writable);
```

## a. What happens in background? (simple)

1. readable emits `"data"`  
2. pipe listens for chunks  
3. pipe calls `writable.write(chunk)`  
4. if writable is slow → `write()` returns **false**  
5. pipe **pauses** readable  
6. writable fires `"drain"` when ready  
7. pipe **resumes** readable  
8. when readable ends → pipe calls `writable.end()`  

## b. When should you use `pipe()`?

Use pipe() when:

- copying files  
- downloading → saving  
- uploading → processing  
- compressing files  
- streaming logs  
- proxying HTTP requests  

---

## 6️⃣ File System (fs) 

The `fs` module in Node.js allows you to interact with the computer’s file system. It lets you read files, write files, delete files, create folders, list folders, stream large files, and work with file metadata — all using JavaScript.

Just like streams and buffers help Node process data efficiently, the file system module lets Node talk to your disk without blocking the main thread. Most operations in `fs` have two versions:

1. **Synchronous (blocking)** → `fs.readFileSync()`
2. **Asynchronous (non-blocking)** → `fs.readFile()`

As a backend developer, always prefer **asynchronous versions** because they allow Node to handle more requests without freezing.

## 1. Why File System Exists 

Node.js runs outside the browser, so it needs direct access to the OS — especially the file system, which stores:

- configuration  
- logs  
- temporary files  
- data exports  
- uploaded images/videos  
- static content  

The `fs` module is the bridge between Node.js and the operating system’s disk.

## 2. How Node Reads a File Internally

When you run:

```js
fs.readFile("a.txt", "utf8", (err, data) => {
  console.log(data);
});
```

What happens internally:

- JavaScript registers the read operation.  
- Hands it to **libuv** (NOT V8).  
- libuv requests the OS to read the file.  
- File is read into **buffers** behind the scenes.  
- When finished, your callback is executed.  
- Data is returned as string (if encoding is set) or Buffer.

## 3. Reading Files

### a. Asynchronous (Recommended)
```js
const fs = require("fs");

fs.readFile("data.txt", "utf8", (err, data) => {
  if (err) throw err;
  console.log("File content:", data);
});
```

### b. Synchronous (Blocking)
```js
const data = fs.readFileSync("data.txt", "utf8");
console.log(data);
```

### c. Reading as Buffer
```js
fs.readFile("image.png", (err, data) => {
  console.log(data); // <Buffer ... >
});
```

## 4. Writing Files

### a. Create or Replace File
```js
fs.writeFile("note.txt", "Hello!", (err) => {
  if (err) throw err;
});
```

### b. Append to File
```js
fs.appendFile("note.txt", "More text\n", (err) => {
  if (err) throw err;
});
```

## 5. File Metadata (Stats)

```js
fs.stat("a.txt", (err, stats) => {
  console.log(stats);
});
```

Useful methods:

- `stats.size`
- `stats.birthtime`
- `stats.mtime`
- `stats.isFile()`
- `stats.isDirectory()`

## 6. Working With Directories

### a. Create Folder
```js
fs.mkdir("myfolder", (err) => {
  if (err) throw err;
});
```

### b. Create Nested Folders
```js
fs.mkdir("a/b/c", { recursive: true }, (err) => {
  if (err) throw err;
});
```

### c. Read Folder (readdir)
Folder contains:
```
a.txt
b.txt
img.png
```

```js
fs.readdir("./", (err, files) => {
  console.log(files);
});
```

Output:
```
[ 'a.txt', 'b.txt', 'img.png' ]
```

### d. Remove Folder
```js
fs.rmdir("myfolder", (err) => {
  if (err) throw err;
});
```

## 7. Deleting Files

```js
fs.unlink("a.txt", (err) => {
  if (err) throw err;
});
```

## 8. Renaming Files

```js
fs.rename("old.txt", "new.txt", (err) => {
  if (err) throw err;
});
```

## 9. Streaming Large Files (createReadStream)

```js
const stream = fs.createReadStream("big.txt", "utf8");

stream.on("data", (chunk) => {
  console.log("Chunk:", chunk);
});

stream.on("end", () => {
  console.log("Done!");
});
```

## 10. Streaming Write (createWriteStream)

```js
const ws = fs.createWriteStream("copy.txt");

ws.write("Hello\n");
ws.write("World\n");
ws.end();
```
## 11. Copying Files (Efficient)

```js
fs.createReadStream("a.txt").pipe(
  fs.createWriteStream("b.txt")
);
```

## 12. Checking If File Exists

### a. Old way
```js
fs.existsSync("a.txt");
```

### b. Better way
```js
fs.access("a.txt", (err) => {
  if (err) console.log("Not found");
});
```

## 13. Watching Files (Auto Reload)

```js
fs.watch("log.txt", () => {
  console.log("File changed!");
});
```

## 14. Extra Useful Methods

### a. readFile with try/catch
```js
async function read() {
  try {
    const data = await fs.promises.readFile("file.txt", "utf8");
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
```

### b. writeFile with promises
```js
await fs.promises.writeFile("output.txt", "Hello!");
```

### c. copyFile
```js
fs.copyFile("a.txt", "b.txt", () => {});
```

---

