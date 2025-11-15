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

## 3️⃣ Event Loop (Clean Definition)
The Event Loop is the mechanism that decides:

- what JS code runs now  
- when async callbacks run  
- when microtasks run  
- when timers execute  
- when I/O events execute  

It enables Node.js to be **non-blocking**, even though JavaScript is single-threaded.

➡ **(Explained in detail above)**

---

