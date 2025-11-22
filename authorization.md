## 1️⃣ Authorization: Bearer <token>

When a user interacts with a modern application like Netflix, Amazon, Twitter, or Meta, the backend must know who the user is before allowing access to protected resources. This identification happens through the `Authorization` header, which is part of every HTTP request the client sends. Unlike public routes such as login, signup, or browsing public content, protected routes like updating a profile, creating a post, placing an order, or fetching private data require a verified identity. To achieve this, APIs use the format `Authorization: Bearer <token>`. Here, “Authorization” is the header name, “Bearer” is the token type, and `<token>` is usually a JWT (JSON Web Token) given to the user after successful login. The term “Bearer” means that whoever holds the token is treated as authenticated, similar to how possessing a valid event ticket allows entry.

When a user logs in, the backend checks the email/password and, if valid, creates a JWT containing safe user details like `id`, `email`, issued time, and expiration time. This JWT is cryptographically signed using a secret key known only to the server. The client then stores the token in localStorage, cookies, or memory. Whenever the user accesses a protected API, the client must send this token inside the Authorization header: `Authorization: Bearer eyJhbGciOiJIUz...`. The backend receives this request, extracts the token by splitting the header value at the space, and passes it to `jwt.verify()`, which checks both the signature and expiration using the same secret key. If the token is valid and not expired, the server decodes it and attaches the payload (user information) to `req.user`, allowing the request to proceed. If the token is missing, the server returns `401 Unauthorized` with “No token provided.” If the token is expired, modified, or signed with a different secret, verification fails and the backend responds with “Invalid token.” Since any modification breaks the signature, JWTs cannot be tampered with.

This system is stateless, meaning the server does not store user sessions; it simply verifies tokens. This makes JWT perfect for mobile apps, large-scale systems, microservices, and distributed environments. A helpful analogy is a nightclub: during login, the user shows ID and receives a wristband (JWT). Later, to enter various zones, the user just shows the wristband. The bouncer (auth middleware) checks if the wristband is real or expired—if valid, the user is allowed in; if not, they are denied. The server behaves the same way by verifying tokens instead of checking the database every time.

Below is the code involved in generating and verifying JWTs. On login, the server signs the token:

```js
const token = jwt.sign(
  { id: user._id, email: user.email },
  "SECRET123",
  { expiresIn: "1h" }
);
```

If anything goes wrong, the server rejects the request with a 401 response. In summary, Authorization: Bearer <token> means: “Here is my authentication token. Verify it and allow access if valid.” It is the backbone of modern API authentication used across all major companies due to its speed, security, scalability, and simplicity.

---
