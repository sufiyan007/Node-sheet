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
