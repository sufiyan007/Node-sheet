# Pagination Interview Questions & Answers

A complete set of interview questions and answers with explanations and examples.

---

## 1. What is pagination and why is it used?

Pagination divides large datasets into smaller chunks so API responses stay fast, light, and efficient. Without pagination, returning thousands of rows slows the database, increases memory usage, increases network load, and may even freeze clients.

---

## 2. What happens if we don’t use pagination?

- Database overload  
- Very slow queries  
- High RAM usage  
- App UI freezes  
- Timeouts  
- Server crashes  

Example:  
`GET /products` returning 1 million rows → browser dies.

---

## 3. What is OFFSET/LIMIT pagination?

OFFSET skips N rows, LIMIT fetches the next X rows.

SQL:
```sql
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;
```

---

## 4. Why is OFFSET pagination slow?

Because the DB must scan and skip all OFFSET rows before returning results. For OFFSET 100000, DB still reads 100000 rows internally.

---

## 5. What is the “inconsistency problem” with OFFSET?

If new rows are inserted, OFFSET will give wrong results.

Example:
Page 1 returns 5 items: `[1,2,3,4,5]`  
Two new items arrive: `[100,101]`  
Page 2 OFFSET 5 → returns `[6,7,8,9,10]`  
User misses `100` and `101`.

---

## 6. What is Keyset / Seek pagination?

Instead of using OFFSET, you fetch based on the last item’s position.

Example:
```sql
SELECT * FROM products WHERE id > lastId ORDER BY id LIMIT 20;
```

---

## 7. Why is Keyset pagination faster?

Because the database does not skip any rows. It uses indexed WHERE conditions instead of scanning.

---

## 8. Why can't Keyset jump to page=50?

Keyset uses positions, not page numbers. You can only go “next” or “previous”, not random access.

---

## 9. What is Cursor Pagination?

Cursor = an encoded bookmark that tells the server where the last page ended.  
Client sends the cursor → server decodes → fetches next set.

Cursor pagination = Keyset + encoding.

---

## 10. Why is cursor better than offset?

- Offset skips rows → inconsistent  
- Cursor tracks last item → always correct  
- Cursor is stable even if data changes  
- Faster for large tables  

---

## 11. What is inside a cursor?

Usually:
- last item’s ID  
- timestamp  
- relevance score (optional)

Example:
```json
{ "id": 500, "createdAt": "2025-11-17T12:00" }
```

---

## 12. How do we encode a cursor?

```js
Buffer.from(JSON.stringify(obj)).toString("base64");
```

---

## 13. How does the server decode it?

```js
JSON.parse(Buffer.from(cursor, "base64").toString("utf8"));
```

---

## 14. Why should cursor be opaque?

Clients should not modify, decode, or assume internal structure.  
Server controls everything.

---

## 15. Example Cursor Pagination Flow

1. Server sends 20 products + nextCursor  
2. Client scrolls  
3. Client sends cursor  
4. Server decodes  
5. Server fetches next 20 items  
6. Server sends new cursor  

---

## 16. Cursor Pagination PostgreSQL Example

```sql
SELECT * FROM products
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

---

## 17. Cursor Pagination MongoDB Example

```js
Product.find({
  $or: [
    { createdAt: { $lt: createdAt } },
    { createdAt, _id: { $lt: id } }
  ]
})
.sort({ createdAt: -1, _id: -1 })
.limit(20);
```

---

## 18. Why do we need stable sorting in cursor pagination?

Sorting must be the same on every request, or cursor results jump and break.

Use:
```
ORDER BY created_at DESC, id DESC
```

---

## 19. What metadata should paginated APIs return?

- `data`
- `nextCursor`
- `hasMore`
- `limit`
- `previousCursor` (optional)

---

## 20. When should we use Offset vs Cursor?

| Use Case | Method |
|---------|--------|
| Admin Panels | Offset |
| Small datasets | Offset |
| Infinite scroll | Cursor |
| Social feeds | Cursor |
| Massive datasets | Cursor |
| Real-time updates | Cursor |

---

# END OF FILE
