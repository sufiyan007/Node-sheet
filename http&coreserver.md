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

### Express (wrapper over http)

Adds routing, middleware, body parsing, utilities like res.json and res.status.

## 4. req & res are Streams — what that means

Streams = data delivered in chunks.

req = Readable stream  
res = Writable stream

Reading raw request body:

```js
let body = '';
req.on('data', chunk => { body += chunk.toString(); });
req.on('end', () => console.log(body));
```

File streaming:

```js
const fs = require('fs');
fs.createReadStream('big.mp4').pipe(res);
```

## 5. Request vs Response — components

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

## 6. Status codes, headers, response body — deep explanation

### Status Codes

- 200 OK
- 201 Created
- 204 No Content
- 400 Bad Request
- 401 Unauthorized
- 403 Forbidden
- 404 Not Found
- 500 Internal Server Error

### Headers

**Request headers:**

- Content-Type
- Accept
- Authorization
- Cookie
- User-Agent

**Response headers:**

- Content-Type
- Content-Length
- Set-Cookie
- Cache-Control
- Location
- CORS headers

Setting headers:

```js
res.setHeader('Content-Type', 'application/json');
res.end(JSON.stringify({ ok: true }));
```

### Response Body Types

- JSON
- HTML
- Text
- Binary/File via streams

## 7. HTTP is stateless — what it means

Each request is independent.  
Server does NOT remember previous requests.

To store state:

- Cookies + session store
- JWT tokens

## 8. Why Express?

- Router
- Middleware
- Body parsing
- Error handling
- Convenience helpers
- Easy integration (cors, helmet)

## 9. Production code examples

### Raw Node example

```js
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
  const parsed = url.parse(req.url, true);

  if (req.method === 'POST' && parsed.pathname === '/echo') {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ received: body }));
    });
  }
});
server.listen(3000);
```

### Express JSON handling

```js
app.use(express.json());

app.post('/api/items', (req, res) => {
  res.status(201).json({ id: 123, ...req.body });
});
```

### Streaming a file

```js
app.get('/download', (req, res) => {
  const stream = fs.createReadStream('big.zip');
  res.writeHead(200, {
    'Content-Type': 'application/zip',
    'Content-Disposition': 'attachment; filename="big.zip"'
  });
  stream.pipe(res);
});
```

## 10. Headers Deep Dive — Details & Gotchas

### Content-Type & parsing

- application/json → express.json()
- application/x-www-form-urlencoded → express.urlencoded()
- multipart/form-data → multer

Wrong header example:

Client sends JSON but sets:

```
Content-Type: text/plain
```

Express won’t parse → you get raw body.

### Cookies

```
Set-Cookie: sid=abc; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=3600
```

### CORS

```txt
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Credentials: true
```

### Content-Length vs Transfer-Encoding

- Content-Length when full body known
- Transfer-Encoding: chunked for streaming

---
