## 1️⃣ Redis

Redis can be imagined as a high-speed memory layer that sits between a Node.js service and a traditional database. Instead of sending every request to a slow data store, the application can operationalize Redis as a temporary storage space where frequently accessed values are kept directly inside memory, enabling very quick lookups and reducing pressure on the core database engine. This improves responsiveness, helps the backend scale to large request volumes, and creates a more seamless user experience under heavy load. In normal workflows, the Node.js service checks Redis first to see if the required information already exists. If Redis has the data, the response returns almost instantly. If Redis does not have the data, the service consults the main database, returns the result to the user, and then writes the same result into Redis for future requests. This caching pattern is heavily adopted inside large distributed platforms because it avoids repetitive queries and optimizes throughput. Redis is also a convenient shared layer for storing user login sessions, short-lived authentication tokens, search results, or other temporary details that do not need permanent storage. Since Redis automatically removes information after a defined expiry time, it becomes extremely useful for session lifecycle management and logout token control.
Below is very simple Node.js code that shows how the application connects to Redis and stores or retrieves values without complexity:

```js
// redisClient.js
import { createClient } from 'redis';

const redisClient = createClient({ url: 'redis://localhost:6379' });

redisClient.on('error', (err) => console.log('Redis error:', err));

export async function startRedis() {
  if (!redisClient.isOpen) {
    await redisClient.connect();
    console.log('Redis connected');
  }
}

export { redisClient };
```
Here is basic usage for saving and reading temporary values:

```js
import { redisClient } from './redisClient.js';

export async function saveUser(id, name) {
  await redisClient.set(`user:${id}:name`, name);
}

export async function readUser(id) {
  return await redisClient.get(`user:${id}:name`);
}
```

This is how session data can be maintained with automatic expiry:

```js
import { redisClient } from './redisClient.js';

export async function saveSession(sessionId, data) {
  await redisClient.set(
    `session:${sessionId}`,
    JSON.stringify(data),
    { EX: 1800 } // 30 minutes
  );
}

export async function readSession(sessionId) {
  const value = await redisClient.get(`session:${sessionId}`);
  if (!value) return null;
  return JSON.parse(value);
}

```

Here is a simple pattern for blocking tokens during logout:

```js
import { redisClient } from './redisClient.js';

export async function blockToken(tokenId, expirySeconds) {
  await redisClient.set(
    `blocked:${tokenId}`,
    true,
    { EX: expirySeconds }
  );
}

export async function isBlocked(tokenId) {
  return await redisClient.exists(`blocked:${tokenId}`);
}

```

And here is how Redis accelerates repeated search requests:
```js
import { redisClient } from './redisClient.js';
import Product from './models/Product.js';

export async function searchProducts(text) {
  const key = `search:${text}`;

  const cached = await redisClient.get(key);
  if (cached) return JSON.parse(cached);

  const products = await Product.find({
    name: { $regex: text, $options: 'i' }
  }).lean();

  await redisClient.set(key, JSON.stringify(products), { EX: 600 });

  return products;
}

```
All of these examples illustrate how Redis functions as a performance multiplier. By storing temporary values inside memory, Redis enables faster reads, reduces latency, simplifies token management, and enhances scalability across multiple backend nodes. This makes Redis a standard capability inside modern system design conversations, especially in high-traffic environments similar to Amazon or Google.

---
