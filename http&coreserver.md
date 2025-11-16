## 1️⃣ HTTP 

## 1. What is HTTP and why is it used?

HTTP = HyperText Transfer Protocol. It’s a text-based application protocol used by clients (browsers, mobile apps, other services) to request resources from servers and by servers to send responses back.

Why we use HTTP:

- Universal: browsers, mobile apps, curl, Postman all speak HTTP.
- Human-readable by design (headers are textual).
- Well-defined semantics (methods like GET/POST/PUT/DELETE) and status codes (200/404/500).
- Works over TCP (reliable, ordered byte stream), optionally secured by TLS (HTTPS).
- Fits REST / resource-oriented APIs very well.

You use HTTP because it’s the standard for web APIs and web pages. Everything else (WebSockets, gRPC, raw TCP) is used when HTTP semantics don’t fit.

## 2. How HTTP communication actually happens (transport & handshake)

HTTP messages travel over TCP.

The client opens a TCP connection to the server’s IP and port (80 for HTTP, 443 for HTTPS).

### HTTPS (HTTP over TLS)

- Client connects to server TCP:443
- TLS handshake
- Server proves identity using certificate
- Client & server agree on encryption keys
- HTTP messages then flow encrypted

## 3. Node.js http module

### Node http module (low-level)

Parses raw HTTP into req and res objects.

```js
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello world');
});
server.listen(3000);
```

## 4. Request vs Response — components

### Request contains:

- Method (GET, POST…)
- URL/path
- Headers
- Body (optional)

### Response contains:

- Status code
- Headers
- Body

Example:

```js
app.post('/api/user', express.json(), (req, res) => {
  res.status(201).json({ id: 1, ...req.body });
});
```

## 5. HTTP is stateless — what it means

Each request is independent.  
Server does NOT remember previous requests.

To store state:

- Cookies (at client side) + session store (server side using DB/Redis) (state)
- JWT tokens (stateless)

---

## 2️⃣ Creating HTTP Server

## 1. What Does “Creating an HTTP Server” Mean?

Creating an HTTP server means your backend application:

- Opens a port (like 3000)
- Listens for incoming HTTP requests
- Parses each request (URL, method, headers, body)
- Processes the request using your logic
- Creates a response (status code, headers, body)
- Sends the response back to the client

Think of it like this:

Client (Browser/App) → sends HTTP request  
Server → listens → processes → responds  
Client → shows result (HTML/JSON/etc.)

Your backend is essentially a listener that keeps running 24/7.

---

## 2. How a Browser Actually Talks to Your Server (Amazon/Flipkart Example)

User searches "iPhone" on Amazon.

### Step 1 — Browser creates an HTTP request

```
GET /search?query=iphone HTTP/1.1
Host: amazon.com
User-Agent: Chrome/124
Accept: text/html
```

### Step 2 — Browser opens a TCP connection  
To amazon.com on port 443 (HTTPS).

### Step 3 — Browser sends the HTTP request packet  
It reaches the load balancer → backend.

### Step 4 — Server receives the request  
Reads method, URL, headers, body.  
Runs logic and decides the response.

### Step 5 — Server sends response

```
HTTP/1.1 200 OK           //status
Content-Type: text/html   // headers
<html>...</html>          // Body
```

### Step 6 — Browser displays result.

### Important clarification  
The backend server **is NOT started per request** it will alway runs then only accept the request.  
It is already running 24/7 on the cloud.

---

## 3a. Routing Manually in Raw Node HTTP and using Express
   
```js
const http = require("http");

const server = http.createServer((req, res) => {
  if (req.method === "GET" && req.url === "/") {
    res.writeHead(200, { "Content-Type": "text/plain" });
    return res.end("Home Page");
  }

  if (req.method === "GET" && req.url === "/products") {
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(JSON.stringify(["iPhone", "Samsung"]));
  }

  res.writeHead(404);
  res.end("Not Found");
});

server.listen(3000);
```

Raw Node has no routing → manually match URLs.
From above it is difficulty to handle the routes and hard to manage/understand the code that/s where come the Express.js

---

## 3b. Creating Server Using Express 

```js
const express = require("express");
const app = express();

app.get("/", (req, res) => {
  res.send("Home Page");
});

app.get("/products", (req, res) => {
  res.json(["iPhone", "Samsung"]);
});

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

**Why Express is better:**

- Built-in routing
- Middleware
- Body parsing
- Helpers (res.json, res.send, res.status)
- Easier scaling & structure

---

## 4. How Server Responds (Status + Headers + Body)

### Raw Node:

```js
res.writeHead(201, { "Content-Type": "application/json" });
res.end(JSON.stringify({ message: "Created" }));
```

### Express:

```js
res.status(201).json({ message: "Created" });
```

---

## 6. Real-World Flow — User Hitting a Backend

User accesses:  
https://flipkart.com/search?q=laptop

Flow:

Browser → DNS → TLS Handshake → Load Balancer → Node Server → Router → Controller → DB → Response → Browser renders.

Your HTTP server is the entry gate.

---

## 3️⃣ URL & Params Parsing (Express.js)

## 1. What Is a URL? (Backend Perspective)

A URL tells the backend what resource the client wants and what details are being passed to process that request.

We use one example everywhere:

`https://amazon.com/products/iphone?color=black&sort=price`

| Part | Meaning |
|------|---------|
| `https://` | Protocol (HTTPS) |
| `amazon.com` | Domain / Host |
| `/products/iphone` | Path (resource route) |
| `?color=black&sort=price` | Query Params |
| `#reviews` | Fragment (never sent to backend) |

### Express URL-related methods

| Method | What it does |
|--------|--------------|
| `req.url` | Full URL (path + query) |
| `req.originalUrl` | URL before middleware changes |
| `req.path` | Path only |
| `req.query` | Query parameters |
| `req.params` | Route/path parameters |

## 2. URL in Express.js

Express splits the URL into meaningful parts automatically.

Example Request:

`GET https://amazon.com/products/iphone?color=black&sort=price`

| Property | What it gives you | Example Value |
|----------|-------------------|----------------|
| `req.path` | The URL path | `/products/iphone` |
| `req.params` | Path params | `{ productName: "iphone" }` |
| `req.query` | Query params | `{ color: "black", sort: "price" }` |
| `req.url` | Path + query | `/products/iphone?color=black&sort=price` |
| `req.originalUrl` | URL before middleware changes | `/products/iphone?color=black&sort=price` |

Example Route:

```js
app.get("/products/:productName", (req, res) => {
  res.json({
    path: req.path,
    params: req.params,
    query: req.query,
    url: req.url,
    originalUrl: req.originalUrl
  });
});
```

## 3. Path / Route Params (req.params)

Path params extract dynamic values directly from the URL.

Example URL: `/products/iphone`

Route:

```js
app.get("/products/:productName", (req, res) => {
  res.send(req.params);
});
```

Result:

```json
{ "productName": "iphone" }
```

Use path params when identifying a specific resource:

- `/products/:productName`
- `/orders/:orderId`
- `/users/:id`

## 4. Query Params (req.query)

Query params provide extra filtering or configuration.

Example:  
`?color=black&sort=price`

Route:

```js
app.get("/products/:productName", (req, res) => {
  res.send(req.query);
});
```

Result:

```json
{
  "color": "black",
  "sort": "price"
}
```

Common use cases:

- filtering  
- sorting  
- pagination  

## 5. Combining Params + Queries

Example URL:  
`/products/iphone?color=black&sort=price`

Example:

```js
app.get("/products/:productName", (req, res) => {
  res.json({
    params: req.params,
    query: req.query
  });
});
```

Output:

```json
{
  "params": { "productName": "iphone" },
  "query": { "color": "black", "sort": "price" }
}
```

## 6. Optional Route Params

Optional params are used when a route may or may not have a parameter.

Route:

```js
app.get("/products/:productName?", (req, res) => {
  res.send(req.params.productName || "No product selected");
});
```

### Explanation of `?`:

- `/products` → parameter missing → returns `"No product selected"`
- `/products/iphone` → parameter present → returns `"iphone"`
- The URL still matches whether the param exists or not.

This is helpful when:

- You want a default fallback value,
- You want same handler for list + detailed view.

## 7. Wildcard Routes (*)

Wildcard routes match anything after a prefix.

Example:

```js
app.get("/products/*", (req, res) => {
  res.send(req.params[0]);
});
```

URL: `/products/images/iphone.jpg`

Output:

```
images/iphone.jpg
```

Useful for:

- Serving assets  
- Catch‑all routes  

## 8. Nested Params / Deep Dynamic URLs

Example URL:  
`/products/iphone/reviews/5`

Route:

```js
app.get("/products/:productName/reviews/:reviewId", (req, res) => {
  res.json(req.params);
});
```

Output:

```json
{
  "productName": "iphone",
  "reviewId": "5"
}
```

## 9. URL Parsing in Express Internally

Example URL:  
`/products/iphone?color=black&sort=price`

Express converts it into:

- `path = "/products/iphone"`
- `params = { productName: "iphone" }`
- `query = { color: "black", sort: "price" }`
- `method = "GET"`
- `url = "/products/iphone?color=black&sort=price"`

## 10. Real‑World Example (Amazon Search)

URL:  
`https://amazon.com/products/iphone?color=black&sort=price`

Route:

```js
app.get("/products/:productName", (req, res) => {
  res.json({
    product: req.params.productName,
    filters: req.query
  });
});
```

Output:

```json
{
  "product": "iphone",
  "filters": {
    "color": "black",
    "sort": "price"
  }
}
```
---

# 🚀 Express Body Parsing & URL Parsing Middlewares 

## 🧩 1. express.json()

Parses JSON request bodies and converts them into a JavaScript object.

| Middleware            | Use Case                              | Example                  | Before Middleware → req.body                                       | After Middleware → req.body                         |
|-----------------------|----------------------------------------|---------------------------|---------------------------------------------------------------------|------------------------------------------------------|
| `express.json()`      | Parse JSON bodies (`application/json`) | `app.use(express.json())` | `'{ "name": "iphone", "color": "black" }'` (string)                 | `{ name: "iphone", color: "black" }` (object)       |


## 🧩 2. express.urlencoded()

Parses HTML form data (URL-encoded), typically from `<form method="POST">`.

| Middleware                               | Use Case                               | Example                                        | Before Middleware → req.body                          | After Middleware → req.body                          |
|-------------------------------------------|------------------------------------------|------------------------------------------------|--------------------------------------------------------|--------------------------------------------------------|
| `express.urlencoded({ extended: true })` | Parse HTML forms (`application/x-www-form-urlencoded`) | `app.use(express.urlencoded({ extended: true }))` | `'product=iphone&color=black'` (string)               | `{ product: "iphone", color: "black" }` (object)      |


## 🧩 3. express.raw()

Reads the body as a **raw Buffer** without converting it.

| Middleware        | Use Case                                 | Example                 | Before Middleware → req.body | After Middleware → req.body        |
|-------------------|-------------------------------------------|--------------------------|-------------------------------|------------------------------------|
| `express.raw()`   | Read binary/raw data (Stripe webhooks, file chunks) | `app.use(express.raw())` | Raw binary stream             | `<Buffer 21 4a 2f ...>` (Buffer)   |


## 🧩 4. express.text()

Reads the body as plain text exactly as sent.

| Middleware        | Use Case               | Example                 | Before Middleware → req.body | After Middleware → req.body                     |
|-------------------|-------------------------|--------------------------|-------------------------------|--------------------------------------------------|
| `express.text()`  | Parse plain text (`text/plain`) | `app.use(express.text())` | `'Hello world'` (string)      | `'Hello world'` (string) *(same, but guaranteed)* |


## 🧩 5. cors()

Enables Cross-Origin Resource Sharing (CORS) so frontend apps on other domains can call your backend.

| Middleware  | Use Case                              | Example            | Before Middleware                | After Middleware                     |
|-------------|----------------------------------------|--------------------|----------------------------------|---------------------------------------|
| `cors()`    | Allow frontend apps from other domains | `app.use(cors())` | Browser blocks request (CORS error) | Browser **allows** request (CORS headers added) |

---

## 4️⃣ HTTP Headers (Request + Response)

## 1. What Are HTTP Headers?

HTTP headers are key-value pairs sent with every request and response.  
They carry **metadata** — information *about* the request/response such as content type, client details, caching rules, cookies, authentication, etc.

Headers DO NOT contain the main response body (HTML/JSON).  
They tell the server/browser *how to interpret* the body.

---

## 2. Request Headers (Sent by Client → Server)

Request headers describe what the client is sending or what it wants.

### Examples & Meaning

### **a. Content-Type**
Tells the server what format the request body is in.

```
Content-Type: application/json
```

Server knows: “Body is JSON, parse it as JSON.”

Other examples:

- `application/x-www-form-urlencoded` (HTML form)
- `multipart/form-data` (file uploads)
- `text/plain`

---

### **b. Accept**
Client tells server which response formats it can accept.

```
Accept: application/json
```

---

### **c. Authorization**
Used for login systems, APIs, JWTs.

```
Authorization: Bearer <token>
```

---

### **d. User-Agent**
Identifies the client app or browser.

```
User-Agent: Mozilla/5.0 Chrome/122
```

### **e. Cookie**
Client sends stored cookies back to the server.

```
Cookie: sessionId=abc123; theme=dark
```

## 3. Response Headers (Sent by Server → Client)

### **a. Content-Type**
Server tells client what type of data is being returned.

```
Content-Type: application/json
```

### **b. Content-Length**
Body size in bytes.

### **c. Set-Cookie**
Server instructs browser to store a cookie.

```
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict
```

### **d. Cache-Control**
Controls caching behavior.

```
Cache-Control: no-cache
```

### **e. Access-Control-Allow-Origin** (CORS)
Allows cross-domain requests.

```
Access-Control-Allow-Origin: https://myfrontend.com
```

## 4. Cookies 

Cookies are small pieces of data stored in the browser.  
They allow HTTP (stateless) to behave statefully.

### Why cookies exist?

HTTP is stateless → server does NOT remember who the client is.  
Cookies allow:

- authentication (session IDs, JWT)
- remembering preferences (theme=dark)
- tracking carts, user choices
- sessions across requests

### How cookies are stored?

1. Server sends **Set-Cookie** header.
2. Browser saves it.
3. Browser automatically sends the cookie back in the **Cookie** header on every request.

### Cookie Example Flow

**Server sends cookie:**

```
Set-Cookie: sessionId=abc123; HttpOnly; Secure; Max-Age=3600
```

**Browser stores it.**

**Browser sends it on every request:**

```
Cookie: sessionId=abc123
```

### Cookie Attributes Explained

| Attribute | Meaning |
|----------|---------|
| `HttpOnly` | JS cannot read the cookie (XSS protection) |
| `Secure` | Cookie only sent over HTTPS |
| `SameSite` | Prevent CSRF (Strict/Lax/None) |
| `Max-Age` | Expiration time in seconds |
| `Path` | Which routes send the cookie |
| `Domain` | Which domain gets the cookie |

### Session Cookie vs Persistent Cookie

- **Session cookies** disappear when browser closes.
- **Persistent cookies** stay until expiration.

### Accessing Cookies in Express

#### Send cookie:

```js
res.cookie("sessionId", "abc123", { httpOnly: true });
```

#### Read cookie:

```js
app.use(require("cookie-parser")());

app.get("/", (req, res) => {
  console.log(req.cookies.sessionId);
});
```

## 5. Combined Example (Request + Response + Cookies)

### Browser sends request:

```
GET /profile
Cookie: sessionId=abc123
Accept: application/json
```

### Server responds:

```
200 OK
Content-Type: application/json
```

Body:
```json
{ "user": "John" }
```

## 6. Summary

- Request headers: client → server metadata.
- Response headers: server → client metadata.
- Cookies allow authentication, sessions, personalization.
- Headers control content type, caching, security, CORS, cookies.

---
5️⃣
6️⃣
7️⃣
8️⃣
