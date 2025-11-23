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
The client stores this token and sends it in the Authorization header. The backend verifies it using middleware:

```js
const token = req.header("Authorization")?.split(" ")[1];
const decoded = jwt.verify(token, "SECRET123");
req.user = decoded;
next();
```
If anything goes wrong, the server rejects the request with a 401 response. In summary, Authorization: Bearer <token> means: “Here is my authentication token. Verify it and allow access if valid.” It is the backbone of modern API authentication used across all major companies due to its speed, security, scalability, and simplicity.

---

## 2️⃣ Refresh Tokens

When a user logs in, the backend creates a JWT access token, which usually expires within a short time like 15 minutes or 1 hour. This is done for security: if the token is stolen, the hacker can only use it for a limited time. But short-lived tokens create a problem—when the token expires, the user would need to log in again, which makes the app frustrating. To solve this, applications use Refresh Tokens. A refresh token is a long-lived, special token given to the user during login along with the access token. While the access token is meant to expire quickly, the refresh token is meant to live much longer—often days or weeks. The refresh token cannot access protected routes; it only has one job: ## request a new access token when the old one expires ##.

Here is how the internal process works. When John logs in, the backend generates two tokens: a short-lived access token and a long-lived refresh token. John stores the access token (usually in memory or localStorage) and stores the refresh token safely (usually in an HTTPOnly cookie so JavaScript cannot steal it). Whenever John uses the app, the access token is sent via the Authorization header. When that access token expires, John’s mobile app or frontend detects the 401 error and automatically sends a second request to the backend with the refresh token, usually in a hidden cookie or a dedicated endpoint like /refresh. The backend verifies the refresh token using a different secret or a different signing key. If it is still valid and not blacklisted, the backend creates a new access token and sends it back without asking John to log in again. This gives the user a seamless experience.

Refresh tokens work because they are not sent on every request—they are stored securely, reducing the chance of theft. They also allow the server to “rotate” sessions without storing anything in memory. If John logs out, the backend can delete or blacklist his refresh token so even if a hacker later tries to use it, the server will reject it. If someone steals John’s access token, it only works briefly. If someone somehow steals his refresh token, it is long-lived, but that’s why secure apps store refresh tokens as HTTPOnly cookies and sometimes tie them to device IDs.

A simple analogy: the access token is like a movie ticket that expires after one show, and the refresh token is like a membership card at the theater. You cannot enter the movie with the membership card alone, but you can use it to get a new ticket without re-verifying your identity every time. The backend behaves the same way: the refresh token cannot directly access protected routes; it only generates a new access token when needed.

Below is the essential backend code for refresh tokens. On login, you generate two tokens:

const accessToken = jwt.sign(
  { id: user._id },
  "ACCESS_SECRET",
  { expiresIn: "15m" }
);

const refreshToken = jwt.sign(
  { id: user._id },
  "REFRESH_SECRET",
  { expiresIn: "7d" }
);


The refresh token is usually stored in a secure HTTPOnly cookie or saved in the database for revocation. When the client’s access token expires, it calls /refresh with the refresh token:

app.post("/refresh", (req, res) => {
  const token = req.cookies.refreshToken;

  if (!token) return res.status(401).json({ msg: "No refresh token" });

  try {
    const decoded = jwt.verify(token, "REFRESH_SECRET");

    const newAccessToken = jwt.sign(
      { id: decoded.id },
      "ACCESS_SECRET",
      { expiresIn: "15m" }
    );

    res.json({ accessToken: newAccessToken });
  } catch {
    return res.status(401).json({ msg: "Invalid refresh token" });
  }
});

---
