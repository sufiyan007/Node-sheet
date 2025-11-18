# REST API Design (FULL COLLAPSIBLE VERSION)

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

CRUD = Create, Read, Update, Delete mapped to REST.

### CRUD Table

| Operation | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| Create | POST | /resources | Creates resource |
| Read All | GET | /resources | List |
| Read One | GET | /resources/{id} | Read single |
| Update | PUT/PATCH | /resources/{id} | Full/partial update |
| Delete | DELETE | /resources/{id} | Remove resource |

### CRUD Example: User API

| Action | Method | Endpoint |
|--------|--------|----------|
| Create User | POST | /users |
| Get All Users | GET | /users |
| Get User | GET | /users/:id |
| Update | PUT/PATCH | /users/:id |
| Delete | DELETE | /users/:id |

</details>

---

<details>
<summary><h2>3️⃣ Idempotency</h2></summary>

Idempotency = calling an operation multiple times results in same state.

### Table

| Method | Idempotent | Explanation |
|--------|------------|-------------|
| GET | Yes | Fetch only |
| PUT | Yes | Replaces resource |
| DELETE | Yes | Resource ends deleted |
| POST | No | Creates new data |
| PATCH | Not always | Depends on logic |

</details>

---

<details>
<summary><h2>4️⃣ Pagination</h2></summary>

Pagination divides large datasets into pages.

---

# 1. Pagination Concept
If DB has 1M products and you call:

```
GET /products
```

→ Slow  
→ Heavy  
→ Client crashes  

Instead use:

```
GET /products?page=1&limit=20
```

---

# 2. Pagination Methods

| Method | How | Pros | Cons | Use |
|--------|-----|------|------|------|
| Offset | Skip rows | Simple | Slow | Small apps |
| Keyset | lastId | Very fast | No jump | Infinite scroll |
| Cursor | Encoded token | Best | Complex | Amazon, Meta |

---

# OFFSET Pagination

Formula:
```
OFFSET = (page - 1) * limit
```

Example: page=5, limit=20 → OFFSET=80

### PostgreSQL
```sql
SELECT * FROM orders ORDER BY id OFFSET 80 LIMIT 20;
```

### Node.js
```js
const offset = (page - 1) * limit;
pool.query("SELECT * FROM orders ORDER BY id OFFSET $1 LIMIT $2",[offset,limit]);
```

### MongoDB
```js
Order.find().skip(skip).limit(limit);
```

---

# KEYSET / SEEK Pagination

Example:
```
GET /products?limit=20&lastId=500
```

Query:
```sql
SELECT * FROM products WHERE id > 500 ORDER BY id ASC LIMIT 20;
```

---

# CURSOR Pagination

Cursor = encoded token telling server where last page ended.

Server returns:

```json
{
  "data": [...],
  "nextCursor": "eyJpZCI6OTgxMjM..."
}
```

Client sends cursor again:

```
GET /products?cursor=eyJpZCI6OTg...
```

Server decodes:

```js
JSON.parse(Buffer.from(cursor,"base64").toString())
```

---

## Why OFFSET breaks  
If new items appear between calls, user misses them.  
Cursor avoids this.

---

## PostgreSQL Cursor Implementation

```js
if (cursor) {
  const d = JSON.parse(Buffer.from(cursor,"base64").toString());
  query = `
    SELECT id,name,created_at
    FROM products
    WHERE (created_at,id) < ($1,$2)
    ORDER BY created_at DESC,id DESC
    LIMIT $3
  `;
  params = [d.createdAt, d.id, limit];
}
```

---

## MongoDB Cursor Implementation

```js
const filter = cursor ? {
  $or: [
    { createdAt: { $lt: createdAt }},
    { createdAt, _id: { $lt: id }}
  ]
} : {};
```

---

<details>
<summary><h3>🔹 Filtering</h3></summary>

Filtering returns only items matching conditions.

Example:
```
GET /products?brand=apple&maxPrice=40000
```

### PostgreSQL
```sql
SELECT * FROM products WHERE brand='apple' AND price<=40000;
```

### MongoDB
```js
const filter = {};
if(req.query.brand) filter.brand=req.query.brand;
if(req.query.maxPrice) filter.price={$lte:Number(req.query.maxPrice)};
Product.find(filter);
```

</details>

---

<details>
<summary><h3>🔹 Sorting</h3></summary>

Sorting arranges data.

| Want | Query |
|------|-------|
| Low→High | `?sort=price` |
| High→Low | `?sort=-price` |

### PostgreSQL
```sql
ORDER BY price DESC
```

### MongoDB
```js
.sort({ price: -1 })
```

</details>

---

<details>
<summary><h3>🔹 Search</h3></summary>

Search finds text matches:

```
GET /search?query=iphone
```

### PostgreSQL Basic
```sql
WHERE name ILIKE '%' || $1 || '%'
```

### PostgreSQL Full Text
```sql
to_tsvector(name) @@ plainto_tsquery($1)
```

### MongoDB
```js
Product.find({ $text:{ $search:req.query.query }});
```

</details>

</details>

---

# END OF FILE
