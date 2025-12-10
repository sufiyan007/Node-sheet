## 1️⃣ Caching with Redis

Caching with Redis means keeping a copy of important data in fast memory so the application does not call the main database every time. Think of the main database as a slow warehouse and Redis as a small fast cupboard right next to your desk. When a request comes to your Node.js server, the server first checks Redis. If the data is present there, the server returns it quickly and does not bother the main database. If the data is missing in Redis, the server goes to the main database, fetches it, returns it to the client, and then stores a copy in Redis for next time. This simple pattern reduces load on the database, lowers response time, and helps the system handle more users.

Inside Redis, cache entries are stored under keys that the backend chooses. A typical pattern is to build keys with prefixes and IDs, such as product:123, user:42:profile, or search:laptop. This naming makes it easy to understand what each key represents and to remove keys when data changes. In large systems, only “hot” data, meaning data that is requested again and again, is usually cached. This includes product details, user profiles, search results, configuration, and any repeated read that is expensive to compute or fetch. Caching is especially important in microservice architectures, where many services might otherwise bombard the same database. Redis provides a central fast layer so all of those services can share cached data.

Here is a very typical Node.js pattern for caching a product by ID using MongoDB as the main database and Redis as the cache. This is often called “cache aside” and it is the most common design:

```js
// productController.js
import { redisClient } from './redisClient.js';
import Product from './models/Product.js'; // Mongoose model

export async function getProduct(req, res) {
  const id = req.params.id;
  const cacheKey = `product:${id}`;

  try {
    // Step 1: check Redis cache
    const cached = await redisClient.get(cacheKey);
    if (cached) {
      const product = JSON.parse(cached);
      return res.json({ source: 'redis', product });
    }

    // Step 2: if not in Redis, get from DB
    const product = await Product.findById(id).lean();
    if (!product) {
      return res.status(404).json({ message: 'Not found' });
    }

    // Step 3: store in Redis with expiry
    await redisClient.set(cacheKey, JSON.stringify(product), { EX: 60 * 5 });

    return res.json({ source: 'database', product });
  } catch (err) {
    console.error('Error in getProduct:', err);
    return res.status(500).json({ message: 'Server error' });
  }
}

```

In this flow, Redis acts as the first stop. The main database is used only when Redis misses the data. This is exactly the pattern that appears again and again in large-scale designs.

---

## 2️⃣ TTL (Time To Live)

TTL, or Time To Live, is the lifetime of a key inside Redis. After TTL time is over, Redis removes the key automatically. This means you can treat Redis like a self-cleaning storage for temporary data. Instead of writing extra code to delete old sessions, old search results, or expired tokens, you simply attach a TTL to the key, and Redis takes care of cleanup. This keeps memory usage under control and avoids bugs where old data sticks around longer than it should.

TTL is especially useful for anything that naturally expires. User sessions expire after some inactivity. Password reset links expire after a few minutes. Search cache can expire after some minutes so that fresh data gets loaded again. Even rate limit counters fit nicely with TTL, because you only care about counts for a short time window. In all of these situations, TTL is like a timer attached to the key. When the timer finishes, the data is gone, and Redis memory is freed.

Here is simple Node.js code that stores a key with TTL, checks remaining TTL, and removes TTL if needed:

```js
// ttlExample.js
import { redisClient } from './redisClient.js';

export async function storeUserProfileWithTTL(userId, profile) {
  const key = `user:${userId}:profile`;
  const ttlSeconds = 60 * 10; // 10 minutes

  await redisClient.set(key, JSON.stringify(profile), { EX: ttlSeconds });
}

export async function getRemainingTTL(userId) {
  const key = `user:${userId}:profile`;
  const secondsLeft = await redisClient.ttl(key);
  return secondsLeft; // -2 = no key, -1 = no TTL, or positive seconds
}

export async function makePermanent(userId) {
  const key = `user:${userId}:profile`;
  await redisClient.persist(key); // remove TTL so key does not expire
}
```

This pattern shows a typical way TTL is used. Data is written with a TTL, and the system can inspect or adjust TTL later if business logic requires it.

---

## 3️⃣ Invalidation

Invalidation is the act of removing or updating cache entries that are no longer correct. Caching is powerful, but it creates a problem: if the original data changes in the database, the old copy in Redis becomes stale and should no longer be used. Invalidation is the process that keeps Redis in sync with the source of truth. If data changes often and cache is not invalidated correctly, users may see outdated values, such as old prices, old profile information, or wrong inventory counts.

The simplest form of invalidation is to delete the cache key when the underlying data changes. For example, when a product is updated in the database, the application also removes product:{id} from Redis. The next time a client asks for that product, the cache miss will cause a fresh fetch from the database, and the updated data will be stored as a new cache entry. This pattern is simple and very effective. In more advanced setups, developers may also choose to update cache immediately instead of deleting it, but the idea is the same: anytime the main data changes, the cached copy must be refreshed or removed.

This Node.js example shows invalidation during an update operation:

```js
// productUpdateController.js
import { redisClient } from './redisClient.js';
import Product from './models/Product.js';

export async function updateProduct(req, res) {
  const id = req.params.id;
  const updates = req.body;
  const cacheKey = `product:${id}`;

  try {
    const product = await Product.findByIdAndUpdate(id, updates, {
      new: true,
      lean: true
    });

    if (!product) {
      return res.status(404).json({ message: 'Not found' });
    }

    // simple invalidation: remove old cache entry
    await redisClient.del(cacheKey);

    // optional: write fresh value back to cache
    await redisClient.set(cacheKey, JSON.stringify(product), { EX: 60 * 5 });

    return res.json({ product });
  } catch (err) {
    console.error('Error in updateProduct:', err);
    return res.status(500).json({ message: 'Server error' });
  }
}
```

In this pattern, cache is always correct because any change in the database is followed by a delete or update in Redis. This is a key concept in discussions around cache correctness.

---

## 4️⃣ Rate Limiting

Rate limiting means controlling how many requests a client can send in a given time window. Redis is perfect for this because it can store counters in memory and attach TTL to them. Each client, often identified by IP address or user ID, has a separate counter. Every incoming request increases the counter. If the counter stays below the allowed limit, the request is allowed. If the counter goes above the limit, the system blocks the request with an error response. When the window ends, TTL causes the counter to disappear, and a new window starts fresh.

This is extremely important in production systems, because without rate limiting, a single buggy client or malicious user could send thousands of requests and overload the backend. With Redis, this control is cheap and fast because counters live in memory and atomic increment operations are built in. For example, you might allow 100 requests per minute for each IP address. Redis keeps track of how many times that IP has accessed the system within the current minute. After one minute, the counter expires automatically due to TTL.

Here is a simple Express middleware for rate limiting using Redis:

```js
// rateLimiter.js
import { redisClient } from './redisClient.js';

export function rateLimiter(windowSeconds, maxRequests) {
  return async function (req, res, next) {
    const ip = req.ip || req.connection.remoteAddress;
    const key = `rate:${ip}`;

    try {
      // increase counter for this IP
      const current = await redisClient.incr(key);

      // if this is the first request in the window, attach TTL
      if (current === 1) {
        await redisClient.expire(key, windowSeconds);
      }

      // if counter goes beyond limit, block the request
      if (current > maxRequests) {
        const retryAfter = await redisClient.ttl(key);
        return res.status(429).json({
          message: 'Too many requests, please try again later',
          retryAfterSeconds: retryAfter
        });
      }

      // within limit, allow request
      next();
    } catch (err) {
      console.error('Rate limiter error:', err);
      // if Redis fails, allow request instead of breaking the app
      next();
    }
  };
}

```
This middleware can be plugged into the Express pipeline at the top, so it protects all routes or specific routes, depending on needs. It is a neat example of Redis plus TTL plus simple counters working together in a production-style feature.

---

 ## 5️⃣ Sessions

Sessions represent the logged-in state of a user. In many applications, once a user signs in, the server assigns a session ID and links it to a set of details such as user ID, roles, and preferences. Redis is a common place to store this session data, because it is fast, centralized, and supports TTL. Storing sessions in Redis allows multiple Node.js instances to share the same session store, which is important when the application runs behind a load balancer. Any server can check Redis to decide if a session is valid.

The normal flow looks like this. When a user logs in successfully, the backend creates a session ID, builds a small object with user details, and writes that object into Redis using the session ID as the key. A TTL is attached to that key to represent session duration, such as thirty minutes or one hour. Every protected request from the client includes the session ID, usually in a cookie or header. The backend reads Redis using that session ID. If the session exists, the user is treated as authenticated. If the session is missing or expired, the user is asked to log in again. This pattern is simple, works well with horizontal scaling, and avoids storing session data in local memory of each Node process.

Here is a minimal example with Node.js and Redis for storing and reading sessions:

```js
// sessionStore.js
import { redisClient } from './redisClient.js';
import { randomUUID } from 'crypto';

export async function createSession(userId) {
  const sessionId = randomUUID();
  const key = `session:${sessionId}`;
  const data = { userId };

  // store session with 30 minutes lifetime
  await redisClient.set(key, JSON.stringify(data), { EX: 60 * 30 });

  return sessionId;
}

export async function getSession(sessionId) {
  const key = `session:${sessionId}`;
  const value = await redisClient.get(key);
  if (!value) return null;
  return JSON.parse(value);
}

export async function destroySession(sessionId) {
  const key = `session:${sessionId}`;
  await redisClient.del(key);
}
```

And here is a simple Express middleware that checks the session:

```js
// authMiddleware.js
import { getSession } from './sessionStore.js';

export async function requireSession(req, res, next) {
  const sessionId = req.headers['x-session-id']; // or from cookie
  if (!sessionId) {
    return res.status(401).json({ message: 'No session' });
  }

  const session = await getSession(sessionId);
  if (!session) {
    return res.status(401).json({ message: 'Session expired or invalid' });
  }

  req.userId = session.userId;
  next();
}
```

With this setup, Redis becomes the single place where session state lives. Any Node.js instance can validate sessions by reading from Redis, which fits perfectly with scalable multi-instance deployments.

---
