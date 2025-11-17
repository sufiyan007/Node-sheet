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


---
