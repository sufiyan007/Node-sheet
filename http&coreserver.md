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

## 3. Node.js http module vs Express

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
HTTP/1.1 200 OK
Content-Type: text/html
<html>...</html>
```

### Step 6 — Browser displays result.

### Important clarification  
The backend server **is NOT started per request**.  
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

3️⃣
4️⃣
5️⃣
6️⃣
7️⃣
8️⃣
