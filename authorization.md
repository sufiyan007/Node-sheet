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

```js
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
```

The refresh token is usually stored in a secure HTTPOnly cookie or saved in the database for revocation. When the client’s access token expires, it calls /refresh with the refresh token:

```js
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

```

---

## 3️⃣ Password Hashing

In every modern application—whether it’s Instagram, Amazon, or a banking app—one of the most critical responsibilities of the backend is protecting user passwords. Passwords are the gateway to the user’s identity, personal data, and financial accounts, so the backend must never store them in plain text. If a hacker somehow gains access to the database, raw passwords would immediately expose every user’s account, and even worse, the attacker could use those same passwords on other websites. To prevent this, systems use password hashing, a cryptographic process that transforms real passwords into irreversible, scrambled strings.

When a user signs up, the backend takes their password and runs it through a hashing algorithm like bcrypt or argon2. These algorithms apply a one-way mathematical function designed so that you can convert a password into a hash, but you can never convert a hash back into the original password. This makes hashing the core defense mechanism of authentication systems. Instead of saving “Sufiyan@123”, the database stores something like:
$2b$10$8xkJwnP4dGuj/UDQnJkO7eSOuHtB.vMnzY.....
This hashed value is completely useless to anyone who steals the data, because even supercomputers cannot recover the original password from it.

But hashing alone isn’t enough. Modern hashing systems add a random, unique string called a salt, which is automatically included by algorithms like bcrypt. The salt ensures that even if two users have the same password, their hashes will be completely different. This protects against rainbow table attacks, where attackers try to match known hashes with known passwords. Because the salt makes every hash unique, rainbow tables become useless. Salting, combined with slow hashing algorithms, ensures that brute-force and GPU attacks become extremely costly for attackers.

When the user later logs in, the backend doesn’t decrypt anything. Instead, it takes the password the user enters, hashes it again using the same hashing algorithm and the stored salt, and compares that new hash with the stored one. If both hashes match, the user is authenticated. At no point does the backend ever know or store the original password; it simply compares hashes. If the password is wrong, the computed hash won’t match the stored hash, and the login fails.

This entire system is stateless, lightweight, and secure. Even if your entire database leaks, attackers gain nothing but cryptographic garbage. This is why bcrypt and argon2 are trusted by companies like Netflix, Meta, Google, and PayPal. The backend protects the user without ever touching the real password.

Below is the Node.js code that handles hashing and comparing passwords using bcrypt, the industry standard.

🔹 Hashing Password During Signup

```js

import bcrypt from "bcrypt";

// Takes user's raw password and hashes it before saving
async function hashPassword(password) {
  const saltRounds = 10; // Higher value = stronger + slower
  const hashed = await bcrypt.hash(password, saltRounds);
  return hashed; // This is what you store in DB
}

```

🔹 Comparing Password During Login

```js
import bcrypt from "bcrypt";

// Compares user entered password with the stored hash
async function verifyPassword(enteredPassword, storedHash) {
  const isMatch = await bcrypt.compare(enteredPassword, storedHash);
  return isMatch; // true or false
}

```

🔹 Complete Signup + Login Example

```js
import express from "express";
import bcrypt from "bcrypt";

const app = express();
app.use(express.json());

// Mock DB
let users = [];

// SIGNUP
app.post("/signup", async (req, res) => {
  const { email, password } = req.body;

  const hashedPassword = await bcrypt.hash(password, 10);

  users.push({
    email,
    password: hashedPassword,
  });

  res.json({ message: "User created" });
});

// LOGIN
app.post("/login", async (req, res) => {
  const { email, password } = req.body;

  const user = users.find((u) => u.email === email);
  if (!user) return res.status(400).json({ message: "User not found" });

  const isValid = await bcrypt.compare(password, user.password);

  if (!isValid)
    return res.status(401).json({ message: "Invalid password" });

  res.json({ message: "Login successful" });
});

app.listen(5000, () => console.log("Server running on Port: 5000"));

```

---

## 4️⃣ Cookies / Sessions

When a user interacts with a web application like Amazon, Flipkart, Netflix, or any typical browser-based system, the backend must maintain the user’s identity across multiple requests. This is because HTTP is a stateless protocol, meaning the server does not remember anything about the client after sending a response. Without a memory mechanism, the user would technically be treated as a “new visitor” on every click—losing login state, cart items, preferences, and even the browsing flow. To solve this, web systems use a combination of cookies and sessions, where the cookie lives on the browser and the session lives on the server. This duo allows a website to “remember” who the user is without exposing sensitive data directly.

A cookie is simply a small piece of text data stored in the user's browser. Whenever the server wants the browser to remember something, it sends a Set-Cookie header, and from that point on, the browser automatically attaches that cookie with every request. Cookies can store small information like dark-mode preference, language selection, A/B testing groups, or importantly, the session identifier that connects the browser to its server-side session. Cookies, however, run a security risk if misconfigured, because they are stored on the client side and can be stolen or manipulated through attacks like XSS. That’s why secure flags are crucial.
Important flags include:
        HttpOnly, so JavaScript cannot read the cookie;
        Secure, so it is transmitted only over HTTPS;
        SameSite, to block CSRF attempts;

Max-Age, controlling how long the cookie stays valid.
With these flags correctly configured, the cookie becomes a safe and reliable transport mechanism for session identification.

Sessions operate on the opposite side—server-side. When a user logs in successfully, the backend creates a session object containing key data such as the user ID, role, cart items, and any additional context required to identify or serve that user. This session object is not stored in the browser; instead, it lives in the server memory or a session store like Redis. The server then generates a unique session ID, stores the full session data mapped to that ID, and sends only this tiny session ID to the browser via a cookie. Whenever the browser makes a new request, it automatically sends the sessionId cookie, and the server looks up the corresponding session data. If found, the server knows exactly who is making the request and continues the user’s flow. If the session is missing or expired, the user is treated as logged out.

This model provides strong security because sensitive user information like passwords, role, or cart contents never travel to the client. Only the session ID—a harmless identifier—is stored in the cookie. If a hacker steals the database, they only get session objects without the ability to connect them to specific browsers. If they steal a cookie that lacks security flags, they might hijack sessions, which is why proper cookie security is mandatory. Logging out is also simple: the backend just deletes the session from storage, and the browser’s cookie becomes useless. This separation of responsibilities—cookies for identity transport and sessions for secure state management—is why cookie-session architecture has remained the backbone of login systems for the last 25 years.

A simple analogy is a hotel system. When you check in, the hotel stores all your booking information in their computer (the session). They then give you a room key card (the cookie). The key card does not store your booking—it simply identifies you. When you swipe it at your door, the hotel system looks up your session. If your booking is valid, the door opens. If the booking is removed (logout), the card stops working. This perfectly mirrors how cookies and sessions work inside a backend system.

To bring everything together, here is the complete Node.js implementation that shows login, session creation, cookie generation, access to protected routes, and logout functionality, demonstrating the full lifecycle.

```js

import express from "express";
import cookieParser from "cookie-parser";
import { v4 as uuid } from "uuid";

const app = express();
app.use(express.json());
app.use(cookieParser());

// basic in-memory store (use Redis in real systems)
const sessionStore = {};

app.use((req, res, next) => {
  const sessionId = req.cookies.sessionId;

  if (sessionId && sessionStore[sessionId]) {
    req.sessionData = sessionStore[sessionId];
  } else {
    req.sessionData = null;
  }

  next();
});

function createSession(userId) {
  const sessionId = uuid();
  sessionStore[sessionId] = {
    userId,
    createdAt: Date.now(),
  };
  return sessionId;
}

// LOGIN
app.post("/login", (req, res) => {
  const { email, password } = req.body;

  if (email === "demo@mail.com" && password === "123456") {
    const sessionId = createSession("USER_001");

    res.cookie("sessionId", sessionId, {
      httpOnly: true,
      secure: true,
      sameSite: "strict",
      maxAge: 1000 * 60 * 60 * 24, // 1 day
    });

    return res.json({ msg: "Logged in" });
  }

  res.status(401).json({ msg: "Invalid credentials" });
});

// PROTECTED ROUTE
app.get("/profile", (req, res) => {
  if (!req.sessionData) {
    return res.status(401).json({ msg: "Not logged in" });
  }

  res.json({
    msg: "Profile data",
    userId: req.sessionData.userId,
  });
});

// LOGOUT
app.post("/logout", (req, res) => {
  const sessionId = req.cookies.sessionId;

  if (sessionId) {
    delete sessionStore[sessionId];
  }

  res.clearCookie("sessionId");
  res.json({ msg: "Logged out" });
});

app.listen(5000, () => console.log("Server running on port 5000"));

```

---

## 5️⃣ OAuth

In modern web and mobile applications, users prefer signing in with Google, GitHub, Facebook, or Apple instead of creating new accounts everywhere. The technology that allows this secure “Login with Google” experience is OAuth. The problem OAuth solves is simple: How can your app use someone’s Google identity without ever seeing their Google password? Before OAuth existed, apps used to ask users directly for their Google or Facebook passwords, which was extremely unsafe. OAuth fixes this by allowing Google to handle the login while your app receives only a safe, temporary token instead of the password.

When the user clicks “Login with Google,” your app does not handle the login itself. Instead, it redirects the user to Google’s official login page, where the user enters their password safely. Then Google asks the user, “Do you allow this app to access your profile and email?” If the user accepts, Google sends your app a short authorization code. This code is not the login—it is just a temporary slip. Your backend then exchanges this code with Google’s servers to receive an access token, which allows your app to fetch the user's profile data. During this entire process, your app never touches the password; it only receives permission to access basic information. This makes OAuth both safe for users and convenient for developers.

To make it clearer, imagine giving a valet your car. You don’t hand the valet your house keys, wallet, or personal stuff—only the car key they need. OAuth works the same way: instead of giving your Google password to an app, you give it a temporary permission slip issued by Google, allowing it to access only what you agreed to. This keeps the user safe and simplifies login for your app.

# Node.js OAuth Example 
Below is a very minimal Google OAuth example using Passport. 

```js
npm install express passport passport-google-oauth20 cookie-session
```

# Server and Basic Setup
```js
import express from "express";
import passport from "passport";
import GoogleStrategy from "passport-google-oauth20";
import cookieSession from "cookie-session";

const app = express();

app.use(
  cookieSession({
    name: "session",
    keys: ["SECRET123"],
  })
);

app.use(passport.initialize());
app.use(passport.session());

passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((user, done) => done(null, user));

```

# Google OAuth Strategy

```js
passport.use(
  new GoogleStrategy(
    {
      clientID: "GOOGLE_CLIENT_ID",
      clientSecret: "GOOGLE_CLIENT_SECRET",
      callbackURL: "/auth/google/callback",
    },
    (accessToken, refreshToken, profile, done) => {
      return done(null, profile);
    }
  )
);

```

# Auth Routes (Login, Callback, Profile, Logout)
```js
app.get(
  "/auth/google",
  passport.authenticate("google", { scope: ["profile", "email"] })
);

app.get(
  "/auth/google/callback",
  passport.authenticate("google", { failureRedirect: "/login" }),
  (req, res) => res.redirect("/profile")
);

app.get("/profile", (req, res) => {
  if (!req.user) return res.status(401).json({ msg: "Not logged in" });
  res.json(req.user);
});

app.get("/logout", (req, res) => {
  req.logout(() => {});
  req.session = null;
  res.json({ msg: "Logged out" });
});

```

---
