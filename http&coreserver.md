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
