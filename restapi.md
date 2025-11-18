# REST API Design 

---

<details>
<summary><h2>1️⃣ REST Principles</h2></summary>

REST (Representational State Transfer) is a style for designing APIs that are simple, predictable, and scalable.

### REST Principles Table

| Principle | Explanation |
|----------|-------------|
| Client–Server | Client handles UI & requests. Server handles data & logic. Both evolve independently. |
| Stateless | Server does not store sessions. Every request carries tokens/params. |
| Uniform Interface | URLs = nouns, Methods = verbs (GET/POST/PUT/PATCH/DELETE), JSON responses. |
| Layered System | Requests may pass through gateways, caches, load balancers. |
| Cacheable | Responses can be cached using `Cache-Control`. |
| Code-On-Demand | Server can send JS (rare). |

</details>

---

<details>
<summary><h2>2️⃣ CRUD API Design</h2></summary>

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

</details>

---

<details>
<summary><h2>3️⃣ Idempotency</h2></summary>

Idempotency = calling an operation multiple times results in same state.

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

</details>

---

<details>
<summary><h2>4️⃣ Pagination</h2></summary>

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

---

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
]

---

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

---

# 5. Cursor

Cursor pagination is the most powerful and modern way of paginating API results. 
Instead of using page numbers (page=1, page=2) or using an ID directly (lastId=500), cursor pagination gives the client a special encoded token called a cursor. 
This token represents the exact position where the last result ended.


Cursor pagination is a technique where the server sends a secure encoded token (called a cursor) to the client after returning a page of results. This cursor represents the exact position of the last item in the current page. When the client wants the next page, it sends this cursor back. The server decodes it, understands where the previous page ended, and continues fetching from that point.

This removes the need for page numbers and prevents problems caused by OFFSET. Cursor pagination is fast, stable, and suitable for large datasets and real-time applications.

# 1. How Cursor Pagination Works

Cursor pagination works by using a **position marker** instead of page numbers.  
When the server sends a page of results, it includes a **cursor**, which represents the last item on that page.

Example flow:

1. Server returns 20 products  
2. Last product has ID = 500  
3. Server generates a cursor by encoding this info  
4. Client requests the next 20 using that cursor  
5. Server decodes cursor → continues where it left off  

Cursor pagination never breaks even if:

- new products are inserted  
- old products are deleted  
- data changes while user is scrolling  

Offset pagination breaks in these scenarios. Cursor pagination does not.

# 2. Why Offset Pagination Breaks

Example:  
Amazon product IDs:

```
1 2 3 4 5 6 7 8 9 10
```

User loads:

```
GET /products?page=1&limit=5 → [1, 2, 3, 4, 5]
```

While browsing, **new items arrive**:

```
100, 101
```

Now DB becomes:

```
100 101 1 2 3 4 5 6 7 8 9 10
```

Next request:

```
GET /products?page=2&limit=5 → OFFSET 5 → [6,7,8,9,10]
```

User **missed** items `100` and `101`.

This is why Amazon, Instagram, Twitter, etc. **never use OFFSET**.

---

# 3. Amazon Example

## User searches "iPhone 14"

Frontend calls:

```
GET /search?q=iphone14&limit=20
```

## Server processes search

1. Runs full-text search  
2. Sorts by relevance + createdAt  
3. Gets last product → assume `{id:98123}`  
4. Encodes this object as Base64 cursor  

Server responds:

```json
{
  "data": [...20 items...],
  "nextCursor": "eyJpZCI6OTgxMjMsImNyZWF0ZWRBdCI6IjIwMjUtMTEtMTdUMTA6MDUifQ=="
}
```

## User scrolls

Frontend calls:

```
GET /search?q=iphone14&cursor=eyJpZCI6OTgxMjM...
```

## Server decodes cursor

```js
const decoded = JSON.parse(
  Buffer.from(cursor, "base64").toString("utf8")
);
```

Decoded result:

```
{id: 98123, createdAt: "2025-11-17T10:05"}
```

Now server knows exactly where to continue.

---

# 4. Cursor Pagination — Client-Side Logic

Frontend (React-style pseudo-code):

```js
async function loadProducts(cursor = null) {
  const url = cursor
    ? `/search?q=iphone14&limit=20&cursor=${cursor}`
    : `/search?q=iphone14&limit=20`;

  const res = await fetch(url);
  const json = await res.json();

  appendToUI(json.data);

  nextCursor = json.nextCursor || null;
}
```

Client rules:
- Never decode cursor  
- Never modify cursor  
- Never assume page numbers  
- Treat cursor as a **black box**  

# 5. Cursor Pagination — Server Side Logic

Server tasks:

1. Decode cursor (if provided)  
2. Convert timestamp+id into Keyset query  
3. Fetch next batch  
4. Create new cursor  
5. Return to client  

Sorting must always be:

```
ORDER BY created_at DESC, id DESC
```

If the sort changes, cursor pagination breaks.

# 6. Cursor Encoding

Encoding is usually:

```
cursor = base64(JSON.stringify({createdAt, id }))
```

Example JSON:

```json
{
  "createdAt": "2025-11-17T10:00:00.000Z",
  "id": "67584930ab12fe01cd904322"
}
```

Encoded cursor:

```
eyJjcmVhdGVkQXQiOiIyMDI1LTExLTE3VDEwOjAwOjAwLjAwMFoiLCJpZCI6IjY3NTg0OTMwYWIxMmZlMDFjZDkwNDMyMiJ9
```

Decode in Node:

```js
const decoded = JSON.parse(
  Buffer.from(cursor, "base64").toString("utf8")
);
```

# 7. PostgreSQL Implementation

## Step 1: Decode

```js
const decoded = cursor
  ? JSON.parse(Buffer.from(cursor, "base64").toString("utf8"))
  : null;
```

## Step 2: Query

```sql
SELECT id, name, created_at
FROM products
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT $3;
```

## Step 3: Node.js

```js
const limit = Number(req.query.limit) || 20;
const cursor = req.query.cursor;

let createdAt = null;
let id = null;

if (cursor) {
  const decoded = JSON.parse(Buffer.from(cursor, "base64").toString("utf8"));
  createdAt = decoded.createdAt;
  id = decoded.id;
}

let query, params;

if (cursor) {
  query = `
    SELECT id, name, created_at
    FROM products
    WHERE (created_at, id) < ($1, $2)
    ORDER BY created_at DESC, id DESC
    LIMIT $3
  `;
  params = [createdAt, id, limit];
} else {
  query = `
    SELECT id, name, created_at
    FROM products
    ORDER BY created_at DESC, id DESC
    LIMIT $1
  `;
  params = [limit];
}

const { rows } = await db.query(query, params);

// Next cursor
let nextCursor = null;
if (rows.length > 0) {
  const last = rows[rows.length - 1];
  nextCursor = Buffer.from(
    JSON.stringify({ createdAt: last.created_at, id: last.id })
  ).toString("base64");
}
```

# 8. MongoDB Implementation

## Step 1: Decode

```js
let createdAt, id;

if (cursor) {
  const decoded = JSON.parse(Buffer.from(cursor, "base64").toString());
  createdAt = new Date(decoded.createdAt);
  id = decoded.id;
}
```

## Step 2: Filter for Keyset

```js
const filter = cursor
  ? {
      $or: [
        { createdAt: { $lt: createdAt } },
        { createdAt, _id: { $lt: id } }
      ]
    }
  : {};
```

## Step 3: Query

```js
const products = await Product.find(filter)
  .sort({ createdAt: -1, _id: -1 })
  .limit(limit);
```

## Step 4: New Cursor

```js
let nextCursor = null;

if (products.length > 0) {
  const last = products[products.length - 1];
  nextCursor = Buffer.from(
    JSON.stringify({ createdAt: last.createdAt, id: last._id })
  ).toString("base64");
}
```

Interview Questions - <a href="https://github.com/sufiyan007/Node-sheet/blob/main/paginationInterviewQ&A%20.md"> LINK </a>

</details>

---

<details>
<summary><h2>5️⃣ Filtering</h2></summary>

Filtering allows the client to request only the subset of data that matches specific conditions. Instead of returning all products, foods, posts, or orders, the server returns only those items that meet the user's selected criteria. This makes APIs faster, lighter, and more useful.


## What Filtering Means

Filtering means narrowing down a large dataset based on conditions like:

- price range  
- category  
- rating  
- brand  
- availability  
- size  
- date range  

Example (Amazon search):
```
GET /products?category=mobile&brand=samsung&maxPrice=30000&minRating=4
```
The client asks only for mobiles from Samsung under ₹30K and rating above 4.

## 🟣 Why Filtering Exists

## 🟧 Client Side Explanation

Client (React, Android, iOS) simply sends query parameters based on user selection.

Example:
```
GET /products?brand=apple&minPrice=50000&maxPrice=90000&minRating=4
```

Client responsibilities:
- send key=value filters  
- preserve filters during pagination  
- do not send unnecessary keys  
- combine with sorting and pagination  

Client does **not** need to understand database logic.

## 🟩 Server Side Explanation

Backend receives:
```
/products?category=mobile&brand=samsung&minPrice=20000&maxPrice=30000
```

Server steps:
1. Read query params  
2. Validate them  
3. Convert into DB filters  
4. Apply filtering → sorting → pagination  
5. Return final dataset  

Backend controls how filters map to DB fields.

## 🟦 PostgreSQL Filtering Example

User selects Samsung mobiles under ₹30K:
```sql
SELECT * FROM products
WHERE category = 'mobile'
AND brand = 'samsung'
AND price <= 30000
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

## 🟩 MongoDB Filtering Example

```js
const filter = {}

if (req.query.category) filter.category = req.query.category;
if (req.query.brand) filter.brand = req.query.brand;

if (req.query.minPrice) {
  filter.price = { ...filter.price, $gte: Number(req.query.minPrice) }
}
if (req.query.maxPrice) {
  filter.price = { ...filter.price, $lte: Number(req.query.maxPrice) }
}

if (req.query.minRating) {
  filter.rating = { $gte: Number(req.query.minRating) }
}

const products = await Product.find(filter)
  .sort({ createdAt: -1 })
  .limit(20)
```
</details>

---

<details>
<summary><h2>6️⃣ Sorting</h2></summary>

Sorting is the process of arranging data in a specific order before sending it to the client. It allows users to view results in a meaningful way—like highest price, lowest price, newest first, or highest rating. Sorting is performed **after filtering** and **before pagination**, so results stay accurate and consistent.

## 🟧 How Sorting Works (Client Side)

Client simply sends a query parameter telling the backend how to sort.

Examples:

| User Selection | Query Param |
|----------------|-------------|
| Price: Low → High | `?sort=price` |
| Price: High → Low | `?sort=-price` |
| Newest First | `?sort=-createdAt` |
| Oldest First | `?sort=createdAt` |
| Highest Rated | `?sort=-rating` |

Client responsibility:
- Only sends sorting instruction  
- Combines sorting with filters and pagination  
- Must keep sort value constant during scrolling (cursor pagination)

Client **does not** sort anything itself.

## 🟩 How Sorting Works (Server Side)

Backend receives something like:

```
GET /products?sort=-price&limit=20&page=1
```

Server must:
1. Parse sort param  
2. Convert to DB syntax  
3. Combine with filters  
4. Combine with pagination  
5. Return sorted dataset  

Sorting MUST occur before pagination.  
Otherwise, pages will shuffle when sorted.

## 🟦 Real Amazon Example — Complete Flow

### 1️⃣ User selects: **Sort by Price → High to Low**

Frontend calls:

```
GET /search?query=iphone14&sort=-price&page=1&limit=20
```

### 2️⃣ Backend receives query:

```js
req.query = {
  query: "iphone14",
  sort: "-price",
  page: "1",
  limit: "20"
}
```

Backend interprets:

- field = price  
- direction = DESC  

### 3️⃣ Backend converts sorting into SQL or Mongo sorting.

### PostgreSQL converts:
```
ORDER BY price DESC
```

### MongoDB converts:
```js
.sort({ price: -1 })
```

### 4️⃣ Backend executes final DB query

### PostgreSQL:

```sql
SELECT id, name, price, created_at
FROM products
WHERE name ILIKE '%iphone14%'
ORDER BY price DESC
LIMIT 20 OFFSET 0;
```

### MongoDB:

```js
Product.find({ name: /iphone14/i })
  .sort({ price: -1 })
  .limit(20)
  .skip(0);
```

###  Database returns sorted rows  
Backend converts results to JSON and sends them.


## 🟨 Sorting Implementation – PostgreSQL + Node.js

### Step 1: Parse the sort parameter

```js
const sortParam = req.query.sort || "-createdAt";

let sortField = "created_at";
let sortOrder = "DESC";

if (sortParam.startsWith("-")) {
  sortField = sortParam.substring(1);
  sortOrder = "DESC";
} else {
  sortField = sortParam;
  sortOrder = "ASC";
}
```

### Step 2: Use it inside SQL

```js
const sql = `
  SELECT id, name, price, created_at
  FROM products
  ORDERORDER BY ${sortField} ${sortOrder}
  LIMIT $1 OFFSET $2
`;
```

### Step 3: Execute

```js
const result = await db.query(sql, [limit, offset]);
res.json(result.rows);
```

## 🟩 Sorting Implementation – MongoDB + Node.js

### Step 1: Build sort object

```js
let sortObj = {};

if (req.query.sort) {
  if (req.query.sort.startsWith("-")) {
    sortObj[req.query.sort.substring(1)] = -1;
  } else {
    sortObj[req.query.sort] = 1;
  }
} else {
  sortObj["createdAt"] = -1;
}
```

### Step 2: Apply to Mongo query

```js
const products = await Product.find(filter)
  .sort(sortObj)
  .limit(limit)
  .skip(offset);
```

---
</details>

--- 

<details>
<summary><h2>7️⃣ Searching</h2></summary>


</details>

---

