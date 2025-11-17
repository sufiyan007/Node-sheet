# Pagination – Complete Interview Questions & Answers  
A fully detailed, end-to-end document covering Offset, Keyset, and Cursor pagination.

---

# 🟦 Basic Pagination Questions

### **1. What is pagination and why do we use it in APIs?**
Pagination divides a large dataset into smaller chunks so APIs return data faster and the UI does not get overloaded.  
Without pagination, a single API request may return thousands or millions of rows, slowing the server, network, and client.

### **2. What problems occur if we don’t use pagination?**
- Very slow queries  
- High RAM/CPU usage  
- Timeouts  
- App freeze or crash  
- Heavy network usage  
- DB lock or overload  
- Users receive too much irrelevant data  

### **3. What is the difference between page-based and limit-based pagination?**
- **Page-based:**  
  ```
  GET /products?page=3&limit=20
  ```
- **Limit-based:**  
  ```
  GET /products?limit=20
  ```

### **4. What are page and limit in pagination?**
- **page** = which group of results you want  
- **limit** = how many items returned per request  

### **5. Why is returning thousands of records in API harmful?**
- DB overload  
- High memory usage  
- Network congestion  
- Client UI freeze/crash  

---

# 🟩 Intermediate Questions (Offset / Limit)

### **6. How does OFFSET/LIMIT pagination work internally?**
OFFSET tells DB how many rows to skip; LIMIT defines how many to return.

### **7. Write a PostgreSQL query using OFFSET and LIMIT.**
```sql
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;
```

### **8. What are the advantages of Offset pagination?**
- Simple to implement  
- Easy to jump to page numbers  

### **9. What are the disadvantages of Offset pagination?**
- Very slow for large offsets  
- Skips rows → inconsistent results  
- Sensitive to inserts/deletes  

### **10. Why does Offset become slow when tables get large?**
DB must scan & skip every row before OFFSET.

### **11. Why does OFFSET skip rows and cause inconsistencies?**
Because OFFSET is blind.  
If new rows are added, pagination shifts incorrectly.

### **12. Give an example where Offset returns wrong or outdated results.**
If new products are added between page 1 and page 2, OFFSET page 2 misses those new items.

---

# 🟧 Keyset / Seek Pagination Questions

### **13. What is Keyset (Seek) pagination?**
Fetches next rows based on last item’s position using indexed fields.

### **14. How is Keyset pagination different from Offset?**
- Keyset uses `WHERE id < lastId`  
- Offset skips rows  

### **15. Why is Keyset pagination faster?**
Uses indexed comparison → avoids scanning skipped rows.

### **16. Why can't you jump to page 50 with Keyset pagination?**
Because it is position-based, not page-based.

### **17. Why does Keyset require stable sorting?**
Stable order ensures consistent continuation:
```
ORDER BY createdAt DESC, id DESC
```

### **18. Show a SQL query for Keyset pagination.**
```sql
SELECT * FROM posts 
WHERE id < 500
ORDER BY id DESC
LIMIT 20;
```

### **19. Real-world use case where Keyset is required.**
Instagram/Twitter feed → infinite scroll.

### **20. Why does Keyset work better for infinite scroll?**
It loads data continuously without skipping rows.

---

# 🟥 Cursor Pagination Questions

### **21. What is cursor pagination?**
A method where server sends an encoded pointer (cursor) marking last item’s position.

### **22. How does cursor solve problems Offset/Keyset cannot?**
It encodes position safely, works even with inserts/deletes.

### **23. What is a “cursor”?**
An encoded JSON object containing last item info:
```json
{ "id": 123, "createdAt": "2025-11-17T10:00" }
```

### **24. Why must a cursor be opaque?**
Clients should not modify or understand it.

### **25. How do we encode a cursor?**
```js
Buffer.from(JSON.stringify(obj)).toString("base64");
```

### **26. What should be stored inside a cursor?**
- id  
- createdAt  
- other stable sort values  

### **27. Why is a cursor better for real-time data?**
New items do not affect previously fetched pages.

### **28. Example of cursor pagination (Amazon product listing).**
Server sends 20 items + cursor for last item.

### **29. Why can’t cursor pagination jump to page numbers?**
It is position-based, not page-based.

### **30. What happens if the client tampers with a cursor?**
Server will reject it or return empty results.

### **31. Show a Node.js snippet that decodes a cursor.**
```js
JSON.parse(Buffer.from(cursor, "base64").toString("utf8"));
```

### **32. Show a Node.js snippet that generates a cursor.**
```js
Buffer.from(JSON.stringify({id, createdAt})).toString("base64");
```

---

# 🟫 Performance Questions

### **41. Why is OFFSET slow in PostgreSQL/MongoDB?**
It must read all skipped rows → heavy cost.

### **42. Why does OFFSET require scanning skipped rows?**
DB cannot jump directly; rows must be read internally.

### **43. How does Keyset reduce read amplification?**
Reads only needed rows → zero skipping.

### **44. How does cursor reduce network load?**
Only sends concise encoded pointer; no duplicated data.

### **45. What indexing is required for effective pagination?**
- Index on `createdAt`  
- Index on `id`  
- Composite index `(createdAt, id)`  

---

# 🟣 Coding Exercises (Most Common in Interviews)

### **51. Implement Offset pagination in Express + PostgreSQL.**
```js
const page = Number(req.query.page) || 1;
const limit = Number(req.query.limit) || 20;
const offset = (page - 1) * limit;

db.query("SELECT * FROM products LIMIT $1 OFFSET $2", [limit, offset]);
```

### **52. Implement Keyset pagination for /posts sorted by createdAt DESC.**
```js
const last = req.query.lastCreatedAt;

db.query(
  "SELECT * FROM posts WHERE createdAt < $1 ORDER BY createdAt DESC LIMIT 20",
  [last]
);
```

### **53. Implement Cursor pagination for /products using { id, createdAt }.**
```js
const cursor = JSON.parse(Buffer.from(req.query.cursor, "base64").toString());
db.query(
  "SELECT * FROM products WHERE (createdAt, id) < ($1, $2) ORDER BY createdAt DESC LIMIT 20",
  [cursor.createdAt, cursor.id]
);
```

### **54. Decode a Base64 cursor and use it.**
```js
const decoded = JSON.parse(Buffer.from(cursor, "base64").toString("utf8"));
```

### **55. Generate a cursor.**
```js
const cursor = Buffer.from(JSON.stringify(obj)).toString("base64");
```

---

# End of File
