# Node.js Error Handling 

## 1 — The Golden Rule (again, short)
**`try/catch` only catches errors that happen while the `try` block is executing.**  
If an error happens later (in a timer, callback, promise microtask, stream, or other async work), the `try/catch` has already finished — so it cannot catch that error.

---

## 2 — How Node executes code (simple visual / timeline)

```
TIME 0:                      CALL STACK
-----------------            ---------------------
| main script runs |  -->    | run()              |
-----------------            ---------------------
                              ^ try/catch lives here

TIME 1+:                     EVENT LOOP QUEUES
-----------------            ---------------------
| timers (setTimeout) |      | microtasks (Promises) |
| IO callbacks (fs/db) |     | event queue (emitters)|
-----------------            ---------------------
```

Important: A callback scheduled now runs *after* the current stack unwinds. So any `try/catch` around scheduling does not cover the callback execution.

---

## 3 — `try/catch`: what it does and why it fails for async

### Synchronous example (works)
```js
try {
  JSON.parse("not json"); // throws immediately
} catch (err) {
  console.log("caught sync error", err.message);
}
```

### Async example (fails)
```js
try {
  setTimeout(() => {
    throw new Error("boom later");
  }, 100);
} catch (err) {
  console.log("never runs");
}
```

**Timeline explanation**
1. `setTimeout` registers the callback and returns immediately.
2. `try` block finishes.
3. After 100ms, timer callback runs on event loop — outside `try`.
4. The thrown error is uncaught → process would crash unless there's a global handler.

---

## 4 — Callback style (`(err, data)`) — why it exists and how to use it

Old Node functions accept callbacks where the first param is an `err`. Example:

```js
const fs = require("fs");

fs.readFile("file.txt", "utf8", (err, data) => {
  if (err) {
    console.error("Read failed:", err.message);
    return;
  }
  console.log("Contents:", data);
});
```

Why? Because `fs.readFile` is asynchronous. The callback runs later, after the scheduling call has returned. `try/catch` cannot reach the callback; therefore error is delivered as the first argument.

**Mistake to avoid**: forgetting to check `err` leads to silent crashes or undefined behavior.

---

## 5 — Promises

### 5.1 `.then()` and `.catch()`
```js
Promise.resolve()
  .then(() => {
    throw new Error("fail in then");
  })
  .catch(err => {
    console.log("caught:", err.message);
  });
```

`.catch()` handles rejected promises during the microtask phase.

### 5.2 `Promise.reject()` without `.catch()` -> unhandledRejection

```js
Promise.reject("no one handled me");
// Node will emit "unhandledRejection" event (unless handled later)
```

If you add `.catch()` anywhere in the chain before the microtask runs, it's handled.

### 5.3 Example: async code that escapes try/catch

```js
try {
  Promise.reject(new Error("oops"));
} catch (e) {
  console.log("nope");
}
// At this point Node will warn about unhandledRejection
```

Because the rejection happens asynchronously after `try` ended.

---

## 6 — `async` / `await`

`await` pauses execution inside an `async` function. So `try/catch` around `await` works:

```js
async function main() {
  try {
    const val = await Promise.reject(new Error("rejected"));
  } catch (err) {
    console.log("caught via await:", err.message);
  }
}
main();
```

**Key:** `await` makes JS wait at that line for the Promise result — the `try` scope is still active.

---

## 7 — `unhandledRejection` and `uncaughtException` (deep, with reproductions)

### 7.1 `unhandledRejection`
- **When**: A Promise is rejected and no `.catch()` (or try/catch with `await`) handles it before the rejection is processed.
- **What Node does**: Emits an `unhandledRejection` event on `process`. Prior to Node v15 it was a warning; in future versions it's recommended to treat it as fatal (so apps should exit).

**Repro example**
```js
// save as unhandledRejection.js
Promise.reject(new Error("unhandled!"));

process.on("unhandledRejection", (reason, promise) => {
  console.error("Global unhandledRejection:", reason);
  // recommended: log + graceful shutdown
});
```

Run `node unhandledRejection.js`

**Note**: If you later attach `.catch()` to the same promise (within the same tick), Node may consider it handled — but don't rely on late handlers.

### 7.2 `uncaughtException`
- **When**: A synchronous exception is thrown and not caught anywhere — including thrown inside a timer callback, event handler, or any non-promise callback.
- **What Node does**: Emits `uncaughtException`. Historically, apps can catch it, but best practice is to log and exit — because the process state may be corrupted.

**Repro example**
```js
// save as uncaughtException.js
setTimeout(() => {
  throw new Error("boom!");
}, 100);

process.on("uncaughtException", (err) => {
  console.error("Global uncaughtException:", err);
  // do NOT attempt to continue like nothing happened
  process.exit(1);
});
```

Run `node uncaughtException.js`

### 7.3 Best practice for both events
- Log enough info (error, stack, context, request id if web request).
- Gracefully shutdown: stop accepting new requests, finish in-flight requests (with timeout), close DB connections, flush logs, then `process.exit(1)` or let your process manager (PM2/systemd) restart it.
- Do NOT keep process running in the same state — memory/handles may be inconsistent.

---

## 8 — Streams & EventEmitter errors

### 8.1 Streams
Streams emit `'error'` events — these must be handled or they will crash:

```js
const fs = require("fs");
const r = fs.createReadStream("no_file.txt");
r.on("data", d => {});
r.on("error", err => console.log("stream error:", err.message));
```

If you don't listen to `'error'`, Node will throw and potentially crash (`Unhandled 'error' event`).

### 8.2 EventEmitter errors
Any EventEmitter that doesn't handle `'error'` will throw a fatal error if `'error'` is emitted.

```js
const EventEmitter = require('events');
const e = new EventEmitter();

e.emit('error', new Error('boom')); // if no 'error' listener -> crash
```

**Conclusion**: For streams and emitters, always attach `.on('error', ...)` or handle via piping with error handling.

---

## 9 — Database connection errors (MongoDB/Redis/Kafka etc.)

DB clients typically emit errors on their own connections. Examples:

- Mongoose emits `'error'` and `'disconnected'` events.
- Redis client emits `'error'`.

**Typical pattern**
- Listen to connection-level errors and attempt reconnection.
- If DB disconnects, application should either queue requests or return 5xx until connection is restored.
- On unrecoverable errors, shutdown gracefully so process manager restarts.

**Example (mongoose)**
```js
const mongoose = require('mongoose');

mongoose.connect('mongodb://localhost:27017/test')
  .then(()=> console.log('connected'))
  .catch(err => console.error('initial connect failed', err));

mongoose.connection.on('error', err => {
  console.error('MONGO ERROR', err);
  // decide: reconnect, alert, or shutdown
});

mongoose.connection.on('disconnected', () => {
  console.error('MONGO DISCONNECTED');
});
```

DB errors often bypass try/catch because they occur in the driver's internal callbacks — you must handle connection events.

---

## 10 — Express: routes, middleware, and error handling

### 10.1 Basic error flow in Express
- Route code can `throw` or `next(err)` to hand off errors.
- Async route handlers that return a rejected promise must be handled — Express (v4) does not auto-catch rejected promises (v5 will).
- Error middleware signature: `(err, req, res, next)`

### 10.2 Example: regular route + error middleware

```js
const express = require('express');
const app = express();

app.get('/sync', (req, res) => {
  throw new Error('sync error'); // Express will catch and route to error middleware
});

// async route without wrapper (dangerous)
app.get('/async', async (req, res) => {
  // if this throws, Express v4 won't catch it unless you call next(err)
  const data = await someAsync(); // if this rejects, process may crash
  res.json(data);
});

// safe async with try/catch and next
app.get('/safe-async', async (req, res, next) => {
  try {
    const data = await someAsync();
    res.json(data);
  } catch (err) {
    next(err);
  }
});

// error middleware (must be after routes)
app.use((err, req, res, next) => {
  console.error('API ERROR:', err);
  res.status(500).json({ status: 'error', message: err.message });
});

app.listen(3000);
```

### 10.3 Cleaner pattern: `asyncHandler` wrapper
```js
const asyncHandler = fn => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// usage
app.get('/clean', asyncHandler(async (req, res) => {
  const data = await someAsync();
  res.json(data);
}));
```

This automatically catches rejected promises and calls `next(err)`.

---

## 11 — Best practices: logging, graceful shutdown, restart strategy

### 11.1 Logging
- Use structured logs (JSON) with timestamps and request IDs.
- Log stack traces for uncaught errors.
- Integrate with Sentry/Datadog/Logstash.

### 11.2 Graceful shutdown on fatal errors
**Why**: After an uncaught exception/unhandled rejection, the process state may be unreliable. Graceful shutdown helps close resources cleanly and lets a process manager restart the app.

**Pattern**
1. Stop accepting new HTTP connections (`server.close()`).
2. Wait for ongoing requests to finish (with timeout).
3. Close DB connections.
4. Flush logs.
5. `process.exit(1)`.

**Example**
```js
const server = app.listen(3000, () => console.log('listening'));

const shutdown = async (err) => {
  try {
    console.error('Shutting down...', err && err.stack);
    server.close(() => console.log('HTTP server closed'));
    // close DB etc, with await
    // give some time then exit
    setTimeout(() => process.exit(1), 5000);
  } catch (e) {
    console.error('Error during shutdown', e);
    process.exit(1);
  }
};

process.on('uncaughtException', err => {
  console.error('uncaughtException', err);
  shutdown(err);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('unhandledRejection', reason);
  shutdown(reason);
});
```

### 11.3 Restarting
Use a process manager:
- `systemd`, `pm2`, `forever`, or Kubernetes. Let the manager restart on crash.

---

## 12 — Cheat sheet: copy into every Node project

```js
// 1. Basic global handlers
process.on('unhandledRejection', (reason) => {
  console.error('UNHANDLED REJECTION', reason);
  // option: graceful shutdown
});

process.on('uncaughtException', (err) => {
  console.error('UNCAUGHT EXCEPTION', err);
  // option: graceful shutdown
});

// 2. asyncHandler
const asyncHandler = fn => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// 3. Express error middleware (put last)
app.use((err, req, res, next) => {
  console.error(err.stack || err);
  res.status(500).json({ error: err.message || 'Internal server error' });
});

// 4. Streams: always listen to 'error'
stream.on('error', err => console.error('stream err', err));

// 5. DB: handle connection errors
db.on('error', e => console.error('DB error', e));
```

---

## 13 — Practice exercises (do these to internalize)

1. Create a small server that does `setTimeout(() => { throw new Error('t'); }, 100)` and observe behavior with and without `process.on('uncaughtException')`.
2. Make a promise `Promise.reject('x')` and observe `unhandledRejection`. Then attach `.catch()` and see difference.
3. Build an Express endpoint that calls a function returning a rejected promise without try/catch or `asyncHandler` — see what happens.
4. Use `fs.createReadStream` on a non-existent file and handle `'error'` event.
5. Simulate a DB disconnect (stop local Mongo) while app is running and see how `mongoose.connection.on('disconnected', ...)` fires.

---

## 14 — Appendix: Promise combinators quick examples

```js
const u1='https://jsonplaceholder.typicode.com/posts/1';
const u2='https://jsonplaceholder.typicode.com/users/1';

// Promise.all -> rejects if any fail
Promise.all([fetch(u1), fetch(u2)])
  .then(results => Promise.all(results.map(r => r.json())))
  .catch(e => console.error("one failed", e));

// Promise.allSettled -> returns all results (fulfilled + rejected)
Promise.allSettled([fetch(u1), fetch('bad')])
  .then(res => console.log(res));

// Promise.race -> first settled (success or fail)
Promise.race([fetch(u1), fetch(u2)])
  .then(r => console.log('first done', r));

// Promise.any -> first fulfilled
Promise.any([fetch('bad'), fetch(u2)])
  .then(r => console.log('first successful', r))
  .catch(e => console.error('all failed', e));
```

---

## Final notes (practical advice)
- Use `asyncHandler` and centralized error middleware for Express apps.  
- Use `process.on` handlers **only** to log and trigger graceful shutdown — then restart via PM2/systemd/Kubernetes.  
- Do not try to keep running after an `uncaughtException`; restart.  
- Always handle stream and EventEmitter `'error'` events explicitly.  
- Use structured logs + monitoring (Sentry) so you can track and fix issues early.

---

## That's it — you should now:
- Understand why `try/catch` doesn't catch all errors, visually and practically.  
- Know when to use `.catch()`, when to `await` with try/catch, and when global handlers are needed.  
- Have a production-ready pattern for error handling + graceful shutdown.

Happy to:
- Convert this to a single-file PDF for you.  
- Add diagrams (SVG/PNG) and more examples.  
- Create a mini repo with runnable examples (with `package.json` and scripts).

Say what you want next: **"Make repo"**, **"Add diagrams"**, **"Create PDF"**, or **"Short cheatsheet"**.

