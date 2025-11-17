## REST API Design

## 1️⃣ REST Principles 

REST (Representational State Transfer) is a style for designing APIs that are simple, predictable, and scalable. These principles guide how client applications interact with server resources over HTTP.

## REST Principles Table

| Principle | Explanation |
|----------|-------------|
| Client–Server | The system is split into a client (app/web) and a server (backend). The client only handles UI and requests, while the server handles data and logic. This separation allows both sides to evolve independently and scale better. |
| Stateless | The server does not store client sessions in memory. Each request carries everything needed (such as authentication tokens or parameters). This makes the system more reliable and easier to scale horizontally. |
| Uniform Interface | All resources follow a consistent structure. URLs use nouns (/users, /orders), actions use HTTP methods (GET, POST, PUT, PATCH, DELETE), and data is typically returned in JSON. This predictable pattern makes APIs easy to understand and use. |
| Layered System | Requests may pass through layers like load balancers, API gateways, security filters, or caching layers. The client is unaware of these layers, which allows better security, performance, and scaling without changing the API. |
| Cacheable | Some API responses can be stored temporarily by clients, browsers, or CDNs. When responses are marked as cacheable (for example using Cache-Control headers), it reduces server load and improves speed for repeated requests. |
| Code-On-Demand (Optional) | The server may send executable code (like JavaScript) to the client, allowing the client to gain new functionality dynamically. This is rarely used in modern APIs but is still part of REST. |

---

## 2️⃣ CRUD API Design

CRUD represents the four basic operations performed on resources: Create, Read, Update, Delete.  
A well-designed API organizes these operations using clear resource-based URLs and standard HTTP methods to keep the system predictable, simple, and easy to maintain.

## CRUD Operations Table

| Operation | HTTP Method | Endpoint Pattern | Description |
|-----------|-------------|------------------|-------------|
| Create | POST | /resources | Sends data to create a new resource. The server stores it and returns details of the newly created item. |
| Read (List) | GET | /resources | Retrieves a collection of resources. Often used with pagination, filtering, or sorting. |
| Read (Single) | GET | /resources/{id} | Retrieves a specific resource by its unique identifier. |
| Update (Full) | PUT | /resources/{id} | Replaces the entire resource with the new data sent by the client. |
| Update (Partial) | PATCH | /resources/{id} | Updates specific fields of a resource without replacing everything. |
| Delete | DELETE | /resources/{id} | Removes the resource permanently using its identifier. |

## Example: User CRUD API

| Action | HTTP Method | Endpoint | Behavior |
|--------|-------------|----------|----------|
| Create a User | POST | /users | Server creates a new user based on the input data and returns the created record. |
| Get All Users | GET | /users | Returns a list of all users, possibly with filters like `?page=1&limit=20`. |
| Get a Single User | GET | /users/42 | Returns details of the user with ID 42. |
| Update Full User | PUT | /users/42 | Replaces user 42’s data with a new full user object. |
| Update User Partially | PATCH | /users/42 | Updates only selected fields, such as email or name. |
| Delete User | DELETE | /users/42 | Permanently deletes the user record with ID 42. |

---

## 3️⃣ Idempotency

Idempotency describes an operation that produces the same result no matter how many times it is repeated.  
In API design, this ensures that if a request is accidentally sent multiple times (because of network retries, browser refresh, or client bugs), the server should not duplicate actions or corrupt data.

## Idempotent vs Non-Idempotent Methods

| HTTP Method | Idempotent | Explanation |
|-------------|------------|-------------|
| GET | Yes | Fetching data does not change it. Calling it 1 time or 50 times returns the same resource. |
| PUT | Yes | Replaces the full resource. Sending the same payload repeatedly results in the same final state. |
| DELETE | Yes | Deleting the same resource repeatedly ends with the same state: the item is gone. |
| HEAD | Yes | Similar to GET but without response body. |
| OPTIONS | Yes | Provides communication options; calling it repeatedly has no side effects. |
| POST | No | Creates new resources or triggers actions. Repeating it may create duplicates or repeat operations. |
| PATCH | Not guaranteed | Depends on how the server implements partial updates. |

---

## 4️⃣ PAGINATION

# 1. Pagination  

Pagination means dividing a large set of records into smaller pieces so that an API can return data quickly and the client (mobile app or website) can display it without lag or crash. Instead of sending thousands or millions of rows in one response, the server returns the data page by page.

A simple way to understand it: if your database has 1 million products and someone calls

```
GET /products
```

then returning all products at once will overwhelm everything — the database becomes slow, memory shoots up, CPU spikes, network bandwidth increases, and the request can even timeout. The client (React app, Android app, iOS) will freeze because it cannot load such a massive JSON result.

To avoid this, the API only returns a limited chunk, for example:

```
GET /products?page=1&limit=20
```

This returns only 20 products. The next 20 are on page 2, then page 3, and so on. This keeps the server fast, stable, and scalable while ensuring the client loads data instantly.
In real systems like Zomato, Amazon, and Flipkart, pagination is necessary because listing endpoints often contain huge datasets, and no device can load everything at once.

# 2. Pagination Methods 

To achieve pagination, there are three commonly used methods:

| Method          | How it works                      | Pros                           | Cons                               | Usage                                      |
|-----------------|-----------------------------------|--------------------------------|------------------------------------|--------------------------------------------|
| Offset / Limit  | OFFSET = (page - 1) * limit       | Easiest to implement           | Very slow for large OFFSET (skips rows) | Small apps, admin tools                    |
| Keyset / Seek   | Use `WHERE id > last_id`          | Fast at any scale              | Cannot jump to specific pages like page 50 | Infinite scroll, feeds, large tables        |
| Cursor (Opaque) | Server returns an encoded cursor  | Best performance + secure      | More code and logic required       | Professional APIs (Meta, Twitter, Stripe)  |

# 3. OFFSET

We use the formula OFFSET = (page − 1) × limit. The page number tells us which slice the client wants, and the limit tells us how many items each slice contains. So when the client asks for page 5 with a limit of 20, we don’t start from the beginning—we simply skip the first 80 rows and return the next 20. This lets the database hand over just enough data to keep the app fast, light, and scroll-friendly.

🔢 Example: page = 5, limit = 20

Formula:
```
OFFSET = (page - 1) * limit
OFFSET = (5 - 1) * 20
OFFSET = 4 * 20
OFFSET = 80
```

Meaning:

Skip the first 80 rows

Return the next 20 rows (rows 81–100)

So Page 5 contains:

Items 81 → 100

🧩 PostgreSQL Implementation
```
SELECT *
FROM orders
ORDER BY id
OFFSET 80    -- (page-1)*limit
LIMIT 20;    -- limit
```

Backend code (Node.js):
```
const page = Number(req.query.page) || 1;
const limit = Number(req.query.limit) || 20;

const offset = (page - 1) * limit;

const result = await pool.query(
  "SELECT * FROM orders ORDER BY id OFFSET $1 LIMIT $2",
  [offset, limit]
);

res.json(result.rows);
```
🍃 MongoDB Implementation

(Mongo uses .skip() + .limit())
```
const page = Number(req.query.page) || 1;
const limit = Number(req.query.limit) || 20;

const skip = (page - 1) * limit;

const orders = await Order.find()
  .sort({ _id: 1 })
  .skip(skip)    // OFFSET
  .limit(limit); // LIMIT

res.json(orders);
```
📡 How the Client Sends the Request

Frontend makes a simple GET request:
```
GET /orders?page=5&limit=20
```

Example from frontend JS:
```
fetch("/orders?page=5&limit=20")
  .then(res => res.json())
  .then(data => console.log(data));
```

Or using Axios:

```
axios.get("/orders", {
  params: { page: 5, limit: 20 }
});
```

# 4. KEYSET / SEEK

Keyset pagination is a technique where the API fetches the next set of results based on the last item of the previous page instead of skipping rows.  
Instead of calculating pages like page 1, page 2, page 3, it uses a reference point — usually an ID or a timestamp — to continue from where the last page ended.

A simple way to imagine this:  
If you scroll Instagram or YouTube, they don’t use page=1, page=2.  
They load *new results based on the last post you saw.*  
This is exactly what Keyset pagination does.

Example request:

GET /products?limit=20&lastId=500

This tells the server:
“Give me the next 20 products *after the item with ID 500*.”

The query becomes optimized:
```
SELECT * FROM products
WHERE id > 500
ORDER BY id ASC
LIMIT 20;
```

Because the database does NOT skip rows (unlike OFFSET), this method is extremely fast even if you have millions of records.

It works especially well for:
- infinite scroll feeds  
- real-time apps  
- large tables with millions of rows  
- APIs that must stay fast during high traffic  

The limitation is that you cannot “jump” to a page number like page 50, because seek pagination is not page-based — it is position-based.  
But in modern large-scale systems (Zomato, Twitter, Instagram), jumping to page 50 is not needed; users scroll, swipe, or refresh data continuously.

🧩 PostgreSQL Implementation
API Call
```
GET /products?limit=20&lastId=500
```
Code:
```
const limit = 20;
const lastId = 500;

const query = `
  SELECT id, name, price
  FROM products
  WHERE id > $1
  ORDER BY id ASC
  LIMIT $2
`;

const result = await db.query(query, [lastId, limit]);
```

🍃 MongoDB Implementation

```
const limit = 20;

const products = await Product.find({
  _id: { $gt: lastId }
})
.sort({ _id: 1 })
.limit(limit);
```


# 5. Cursor

Cursor pagination is the most powerful and modern way of paginating API results. 
Instead of using page numbers (page=1, page=2) or using an ID directly (lastId=500), cursor pagination gives the client a special encoded token called a cursor. 
This token represents the exact position where the last result ended.

When the client sends the cursor back, the server knows exactly where to continue.

Unlike Offset pagination, cursor pagination does not skip rows.
Unlike Keyset pagination, cursor pagination does not expose internal IDs.
It creates a clean, safe bridge between the client and server.

Cursor pagination is used by:

Instagram API
Meta/Facebook API


Cursor Pagination — Real Meaning

Every page of data ends with a “last item”.
Instead of giving the client the raw ID of that last item, the server gives a secure encoded cursor.

Example server response:
```
{
  "data": [...],
  "nextCursor": "eyJpZCI6IjU2Nzg5YWJjIn0="
}
```

This nextCursor string is an encoded version of:
```
{ "id": "56789abc" }
```

The client then calls:
```
GET /products?cursor=eyJpZCI6IjU2Nzg5YWJjIn0=&limit=20
```

And the server decodes the cursor to know:

The last ID

The last timestamp (optional)

Where to continue fetching from

This way:

The client does not know DB IDs

The server never exposes internal structure

Pagination is stable and safe

Cursor pagination = Keyset pagination + secure cursor encoding + client/server handshake

Why Cursor Pagination Exists 

Because real systems have problems like:

New rows inserted while user is scrolling

Rows deleted while user is scrolling

Users on multiple devices with different timelines

Large datasets where OFFSET kills performance

Cursor pagination solves all of them by remembering where the client stopped, not the page number.

Cursor Pagination — Client Side Explanation
What client receives

Client gets:

data: the page results

nextCursor: a token representing the last item

hasMore: boolean (optional)

next: a link (optional)

What client sends back

Client sends:
```
cursor=<encoded_token>
limit=20
```
What client should NOT do

Do not try to decode cursor

Do not modify cursor

Do not assume it is an ID

Do not assume pagination is page-based

Cursor is a black box to the client.

Cursor Pagination — Server Side Explanation

Server tasks:

Encode last item into a cursor

Send cursor to client

Decode cursor on next request

Fetch next rows using Keyset conditions

Generate next cursor for the next batch

Keep sorting stable

Sorting stability means:

Always sort by a consistent pair like created_at DESC, id DESC

If sorting is unstable, cursor pagination breaks.

Cursor Encoding — How It Works (Very Important)

Cursor encoding = converting meaningful data into a safe, opaque string.

Most common encoding formats:

Base64(JSON) → simplest + readable

JWT without signature → structured

Crypto/AES encoded string → secure

Best practice:
```
cursor = base64( JSON.stringify({ createdAt, id }) )


Example JSON:

{
  "createdAt": "2025-11-17T10:00:00.000Z",
  "id": "67584930ab12fe01cd904322"
}

```
Encoding:
```
eyJjcmVhdGVkQXQiOiIyMDI1LTExLTE3VDEwOjAwOjAwLjAwMFoiLCJpZCI6IjY3NTg0OTMwYWIxMmZlMDFjZDkwNDMyMiJ9
```

Server receives this cursor and decodes it back.

Node.js decode example:
```const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString('utf8'));```


Now the server knows:

lastCreatedAt

lastId

where to continue

Cursor Pagination — PostgreSQL Version
Step 1: Decode cursor
```
const decoded = cursor
  ? JSON.parse(Buffer.from(cursor, "base64").toString("utf8"))
  : null;
```
Step 2: Build query
```
SELECT id, name, price, created_at
FROM products
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT $3;
```
Step 3: Node.js code
```
const limit = Number(req.query.limit) || 20;
const cursor = req.query.cursor;

// Step 1: decode
let createdAt = null;
let id = null;

if (cursor) {
  const decoded = JSON.parse(Buffer.from(cursor, "base64").toString("utf8"));
  createdAt = decoded.createdAt;
  id = decoded.id;
}

// Step 2: build query
let query, params;

if (cursor) {
  query = `
    SELECT id, name, price, created_at
    FROM products
    WHERE (created_at, id) < ($1, $2)
    ORDER BY created_at DESC, id DESC
    LIMIT $3
  `;
  params = [createdAt, id, limit];
} else {
  query = `
    SELECT id, name, price, created_at
    FROM products
    ORDER BY created_at DESC, id DESC
    LIMIT $1
  `;
  params = [limit];
}

const { rows } = await db.query(query, params);

// Step 3: generate next cursor
let nextCursor = null;
if (rows.length > 0) {
  const last = rows[rows.length - 1];
  nextCursor = Buffer.from(
    JSON.stringify({ createdAt: last.created_at, id: last.id })
  ).toString("base64");
}
```

This is a COMPLETE professional cursor pagination.

Cursor Pagination — MongoDB Version

Mongo needs the same process.

Step 1: Decode cursor
```
let createdAt, id;

if (cursor) {
  const decoded = JSON.parse(Buffer.from(cursor, "base64").toString("utf8"));
  createdAt = new Date(decoded.createdAt);
  id = decoded.id;
}
```
Step 2: Build query
```
const filter = {};

if (cursor) {
  filter.$or = [
    { createdAt: { $lt: createdAt } },
    { createdAt, _id: { $lt: id } }
  ];
}
```
Step 3: Fetch data
```
const products = await Product.find(filter)
  .sort({ createdAt: -1, _id: -1 })
  .limit(limit);
```
Step 4: Create next cursor
```
let nextCursor = null;
if (products.length > 0) {
  const last = products[products.length - 1];
  nextCursor = Buffer.from(
    JSON.stringify({ createdAt: last.createdAt, id: last._id })
  ).toString("base64");
}
```

Missing Concepts You Should Know (Important)
1. Cursor is position-based

Never page-based.

2. Cursor works even when rows are inserted or deleted

Because it uses stable sorting (id + createdAt).

3. Cursor must be opaque

Clients should not decode it.

4. Cursor cannot jump to page 50

Same as Keyset.

5. Cursor pagination is best for:

infinite scroll

mobile feeds

huge tables

real-time data streams

low-latency APIs

6. Cursor reduces read amplification

DB reads far fewer rows.

---
