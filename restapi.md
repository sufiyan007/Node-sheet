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
