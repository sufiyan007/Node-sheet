# ⚙️ Node.js — Event Loop & Concurrency (Masterpiece Explanation)

## 🌟 1. What Problem Does the Event Loop Solve?

JavaScript is single-threaded.

➡ It can run only one line at a time  
➡ But Node.js needs to handle thousands of tasks:

- Network requests  
- File reads  
- Database queries  
- Timers  
- Promises  

If JS tried to run all this by itself → server would freeze.

So Node uses something called **Event Loop**.

The event loop allows Node.js to look like it’s doing many things at once, even though JavaScript itself can run only one thing at a time.

### ✨ How?

- JavaScript handles only the logic  
- Heavy work is handed to lower-level systems like **libuv** and the **OS**  
- When they finish, they notify JavaScript through queues  
- The event loop decides:  
  ✔ What should run right now?  
  ✔ What should run after this finishes?  
  ✔ Should microtasks run now?  
  ✔ Did async work finish?  

This gives Node.js **non-blocking behavior**.

---

## 🌟 2. Call Stack — Where Your Code Actually Runs

The call stack is the place where JavaScript executes functions right now.

It is basically:

🗂 A stack of work that JS must execute immediately.

### Example:

```js
function A() { B(); }
function B() { console.log("Hi"); }
A();
```

Execution:

- A() gets pushed  
- Inside A, B() gets pushed  
- console.log runs  
- B() finishes → removed  
- A() finishes → removed  

📌 Only one function runs at a time.  
📌 This is why JavaScript is single-threaded.

---

## 🌟 3. Callback Queue (Macrotask Queue)

This queue stores tasks that should run **after** the call stack is empty.

Contains:

- setTimeout  
- setInterval  
- setImmediate  
- network callbacks  
- file system callbacks  

### Example:

```js
console.log("A");

setTimeout(() => console.log("B"), 0);

console.log("C");
```

Output:

```
A
C
B
```

Because even 0ms timeout is placed into the queue.

---

## 🌟 4. Microtask Queue (Promise Queue)

This is a **high priority** queue.

Contains:

- Promise.then  
- Promise.catch  
- queueMicrotask  

👉 Microtasks always run **before** callback queue.  
👉 And run until the queue is empty.

### Example:

```js
console.log("A");

Promise.resolve().then(() => console.log("B"));

console.log("C");
```

Output:

```
A
C
B
```

---

## 🌟 5. Event Loop Phases (VERY IMPORTANT)

Node.js processes tasks in phases:

### Phase 1 — Timers
Runs callbacks for `setTimeout`, `setInterval`.

### Phase 2 — Pending Callbacks
Runs system-level callbacks.

### Phase 3 — Idle / Prepare
Internal use.

### Phase 4 — Poll Phase
Node waits for:
- IO results  
- File read  
- Network response  
- Incoming data  

### Phase 5 — Check Phase
Runs `setImmediate` callbacks.

### Phase 6 — Close Callbacks
Handles socket close and cleanup events.

📌 After **every** phase, Node checks and runs microtasks first.

---

## 🌟 6. Async Internals — How Node Executes Async Code

Example:

```js
setTimeout(() => console.log("Done"), 2000);
```

Internal flow:

1. JS sees setTimeout  
2. Sends it to **libuv**  
3. JS continues immediately  
4. After 2 seconds, libuv puts callback into queue  
5. Event loop runs it when call stack is empty  

This allows Node.js to:

✔ run thousands of API requests  
✔ handle file operations  
✔ run DB queries  
✔ run timers  
✔ without blocking  

---

## 🌟 7. Combine All Concepts with One Example

```js
console.log("Start");

setTimeout(() => console.log("timeout"), 0);

Promise.resolve().then(() => console.log("promise"));

console.log("End");
```

Execution:

1. Call Stack runs: Start → End  
2. Promise → microtask queue  
3. Timeout → callback queue  
4. Microtasks run first → promise  
5. Then callbacks → timeout  

Final Output:

```
Start
End
promise
timeout
```

---

## 🌟 8. Deep Real Example — 1GB File Read

```js
fs.createReadStream("file")
```

Internal flow:

- libuv reads file in **chunks**  
- each chunk triggers a `"data"` event  
- event loop handles `"data"` callbacks  
- JS processes one chunk at a time  
- `"end"` event fires when done  

✔ Only one chunk is loaded at a time  
✔ Constant memory usage  

---

## 🌟 9. Things You Must Know Before Event Loop

- Sync vs Async  
- Callback vs Promise vs async/await  
- Execution context  
- JavaScript runtime differences  

---

## 🌟 10. Mental Model (Clean & Professional)

**Call Stack**  
➡ where JavaScript runs right now

**Microtask Queue**  
➡ high priority tasks like Promises

**Callback Queue**  
➡ async tasks like setTimeout, IO

**libuv**  
➡ handles async work like file read, network, timers

**Event Loop**  
➡ system that decides which queue to run next

---

## 🌟 FINAL SUMMARY

The Event Loop is the core system in Node.js that decides:

- which code runs now  
- which async callback runs next  
- when microtasks should run  
- when IO operations should run  
- when timers should run  

This mechanism allows Node.js to handle thousands of tasks efficiently while running JavaScript on a single thread.

It is the heart of concurrency in Node.js.

